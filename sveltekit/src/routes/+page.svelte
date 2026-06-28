<script lang="ts">
  import { onMount } from 'svelte';
  import { clearSession, loadSession, saveSession } from '$lib/conversation-store';
  import { clearSystemPrompt, readSystemPrompt, writeSystemPrompt } from '$lib/system-prompt-store';
  import {
    BONSAI_WORKER_URL,
    SUPERTONIC_WORKER_URL,
    PAUSE_MS,
    POST_TTS_COOLDOWN_MS,
    MODELS,
    VOICES,
    SPEECH_BREAK_OPTIONS,
    TTS_LANGUAGE_GROUPS,
    buildWorkerMessages,
    formatWorkerMessagesForDisplay,
    formatPostProcessedTtsText,
    getLlmPromptForTurn,
    splitForSupertonicWithMode,
    DEFAULT_SYSTEM_PROMPT,
    ttsLangForText,
    formatBytes,
    getSpeechRecognition,
    isLikelyEcho,
    messageId,
    modelName,
    turnsToWorkerMessages,
    type BonsaiWorkerMessage,
    type BrowserSpeechRecognition,
    type ConversationTurn,
    type ModelId,
    type SpeechBreakMode,
    type SupertonicWorkerMessage,
    type TtsSegment,
    type TtsLanguage,
    type VoiceId,
  } from '$lib/speech2speech';

  let modelId = $state<ModelId>('4b');
  let voice = $state<VoiceId>('F1');
  let ttsLanguage = $state<TtsLanguage>('na');
  let systemPrompt = $state(DEFAULT_SYSTEM_PROMPT);
  let speechBreak = $state<SpeechBreakMode>('paragraph');
  let turns = $state<ConversationTurn[]>([]);
  let loadedModelId = $state<ModelId | null>(null);
  let ttsReady = $state(false);
  let status = $state('Load models to begin listening.');
  let ttsStatus = $state('Supertonic loads together with Bonsai.');
  let interim = $state('');
  let isListening = $state(false);
  let isLoadingModels = $state(false);
  let isThinking = $state(false);
  let isSpeaking = $state(false);
  let conversationActive = $state(false);
  let error = $state<string | null>(null);
  let tokensPerSecond = $state<number | null>(null);
  let speechSupported = $state(true);
  let speakingTurnId = $state<string | null>(null);
  let sessionHydrated = $state(false);

  const selectedModel = $derived(MODELS.find((m) => m.id === modelId) ?? MODELS[0]);
  const isReady = $derived(loadedModelId === modelId && ttsReady && !isLoadingModels);

  let bonsaiWorker: Worker | null = null;
  let ttsWorker: Worker | null = null;
  let recognition: BrowserSpeechRecognition | null = null;
  let pauseTimer: ReturnType<typeof setTimeout> | null = null;
  let pendingUtterance = '';
  let assistantDraft = '';
  let audioEl: HTMLAudioElement | null = null;
  let recognitionActive = false;
  let isThinkingFlag = false;
  let isSpeakingFlag = false;
  let isReadyFlag = false;
  let activeTurnId: string | null = null;
  let turnsSnapshot: ConversationTurn[] = [];
  let lastAssistantOutput = '';
  let systemPromptSnapshot = DEFAULT_SYSTEM_PROMPT;
  let speechBreakSnapshot: SpeechBreakMode = 'paragraph';
  let ttsLanguageSnapshot: TtsLanguage = 'na';
  let activeTtsLang: TtsLanguage = 'na';
  let speechSegments: TtsSegment[] = [];
  let speechSegmentIndex = 0;
  let segmentPauseTimer: ReturnType<typeof setTimeout> | null = null;
  let responseTextTab = $state<Record<string, 'original' | 'processed' | 'prompt'>>({});
  let resumeListeningAfterSpeak = false;
  let playbackGeneration = 0;
  let persistTimer: ReturnType<typeof setTimeout> | null = null;
  let persistInFlight = false;
  let persistQueued = false;

  $effect(() => {
    turnsSnapshot = turns;
  });

  $effect(() => {
    systemPromptSnapshot = systemPrompt;
  });

  $effect(() => {
    speechBreakSnapshot = speechBreak;
  });

  $effect(() => {
    ttsLanguageSnapshot = ttsLanguage;
  });

  $effect(() => {
    isReadyFlag = isReady;
  });

  $effect(() => {
    isThinkingFlag = isThinking;
  });

  $effect(() => {
    isSpeakingFlag = isSpeaking;
  });

  $effect(() => {
    if (!sessionHydrated) return;
    turns;
    speechBreak;
    modelId;
    voice;
    ttsLanguage;
    schedulePersist();
  });

  function handleSystemPromptInput(event: Event) {
    const value = (event.currentTarget as HTMLTextAreaElement).value;
    systemPrompt = value;
    writeSystemPrompt(value);
  }

  function resetSystemPrompt() {
    systemPrompt = DEFAULT_SYSTEM_PROMPT;
    writeSystemPrompt(DEFAULT_SYSTEM_PROMPT);
  }

  function schedulePersist() {
    if (persistTimer) clearTimeout(persistTimer);
    persistTimer = setTimeout(() => {
      void persistSession();
    }, 300);
  }

  function flushPersist() {
    if (persistTimer) clearTimeout(persistTimer);
    persistTimer = null;
    void persistSession();
  }

  async function hydrateSession() {
    const saved = await loadSession();
    if (saved) {
      turns = saved.turns;
      speechBreak = saved.speechBreak;
      modelId = saved.modelId;
      voice = saved.voice;
      ttsLanguage = saved.ttsLanguage;

      const storedPrompt = readSystemPrompt();
      if (storedPrompt !== null) {
        systemPrompt = storedPrompt;
      } else {
        systemPrompt = saved.systemPrompt;
        writeSystemPrompt(saved.systemPrompt);
      }
    } else {
      const storedPrompt = readSystemPrompt();
      if (storedPrompt !== null) {
        systemPrompt = storedPrompt;
      } else {
        writeSystemPrompt(systemPrompt);
      }
    }
    sessionHydrated = true;
  }

  async function persistSession() {
    if (!sessionHydrated) return;
    if (persistInFlight) {
      persistQueued = true;
      return;
    }
    persistInFlight = true;
    do {
      persistQueued = false;
      await saveSession({
        turns,
        systemPrompt,
        speechBreak,
        modelId,
        voice,
        ttsLanguage,
        updatedAt: Date.now(),
      });
    } while (persistQueued);
    persistInFlight = false;
  }

  function finishLoadingIfReady() {
    if (!isLoadingModels || loadedModelId !== modelId || !ttsReady) return;
    isLoadingModels = false;
    ttsStatus = 'Supertonic ready.';
    if (speechSupported) {
      startConversation();
    } else {
      status = `${modelName(modelId)} + Supertonic ready.`;
    }
  }

  function clearPauseTimer() {
    if (pauseTimer) clearTimeout(pauseTimer);
    pauseTimer = null;
  }

  function startRecognition() {
    if (!recognition || !conversationActive || isThinkingFlag || isSpeakingFlag) return;
    if (recognitionActive) return;
    try {
      recognition.start();
      recognitionActive = true;
      isListening = true;
    } catch {
      recognitionActive = false;
      isListening = false;
    }
  }

  function stopRecognition() {
    clearPauseTimer();
    isListening = false;
    recognitionActive = false;
    try {
      recognition?.stop();
    } catch {
      /* ignore */
    }
  }

  function responseTextTabFor(turnId: string): 'original' | 'processed' | 'prompt' {
    return responseTextTab[turnId] ?? 'original';
  }

  function setResponseTextTab(turnId: string, tab: 'original' | 'processed' | 'prompt') {
    responseTextTab = { ...responseTextTab, [turnId]: tab };
  }

  function assistantTextForTab(turn: ConversationTurn): string {
    const tab = responseTextTabFor(turn.id);
    if (tab === 'processed') {
      return formatPostProcessedTtsText(turn.assistantText, speechBreak);
    }
    if (tab === 'prompt') {
      return getLlmPromptForTurn(systemPrompt, turns, turn.id);
    }
    return turn.assistantText;
  }

  function handleAssistantTextInput(turnId: string, event: Event) {
    const assistantText = (event.currentTarget as HTMLTextAreaElement).value;
    responseTextTab = { ...responseTextTab, [turnId]: 'original' };
    turns = turns.map((turn) => (turn.id === turnId ? { ...turn, assistantText } : turn));
  }

  function clearSegmentPauseTimer() {
    if (segmentPauseTimer) {
      clearTimeout(segmentPauseTimer);
      segmentPauseTimer = null;
    }
  }

  function finishSpeaking() {
    clearSegmentPauseTimer();
    isSpeakingFlag = false;
    isSpeaking = false;
    speechSegments = [];
    speechSegmentIndex = 0;
    speakingTurnId = null;
    status = conversationActive ? 'Listening...' : isReady ? `${selectedModel.name} + Supertonic ready.` : status;
    if (resumeListeningAfterSpeak && conversationActive) {
      setTimeout(startRecognition, POST_TTS_COOLDOWN_MS);
    }
  }

  function stopSpeaking() {
    playbackGeneration += 1;
    clearSegmentPauseTimer();
    audioEl?.pause();
    audioEl = null;
    finishSpeaking();
  }

  function synthesizeCurrentChunk() {
    const segment = speechSegments[speechSegmentIndex];
    if (!segment?.text.trim()) {
      finishSpeaking();
      return;
    }
    const total = speechSegments.length;
    ttsStatus =
      total > 1 ? `Synthesizing part ${speechSegmentIndex + 1} of ${total}...` : 'Synthesizing speech...';
    ttsWorker?.postMessage({
      type: 'speak',
      text: segment.text,
      voice,
      lang: activeTtsLang,
      speed: 1.05,
      steps: 8,
    });
  }

  function scheduleNextSegment(pauseMs: number) {
    clearSegmentPauseTimer();
    if (pauseMs > 0) {
      segmentPauseTimer = setTimeout(() => {
        segmentPauseTimer = null;
        synthesizeCurrentChunk();
      }, pauseMs);
      return;
    }
    synthesizeCurrentChunk();
  }

  function beginSpeaking(text: string, resumeListening: boolean, turnId: string | null = null) {
    clearPauseTimer();
    pendingUtterance = '';
    interim = '';
    stopRecognition();
    audioEl?.pause();
    playbackGeneration += 1;

    activeTtsLang = ttsLangForText(text, ttsLanguageSnapshot);
    speechSegments = splitForSupertonicWithMode(text, speechBreakSnapshot);
    if (speechSegments.length === 0 || !speechSegments.some((segment) => segment.text.trim())) {
      speakingTurnId = null;
      if (resumeListening && conversationActive) {
        setTimeout(startRecognition, POST_TTS_COOLDOWN_MS);
      }
      return;
    }

    lastAssistantOutput = text.trim();
    speechSegmentIndex = 0;
    resumeListeningAfterSpeak = resumeListening;
    speakingTurnId = turnId;
    isSpeakingFlag = true;
    isSpeaking = true;
    synthesizeCurrentChunk();
  }

  function speak(text: string, turnId: string | null) {
    beginSpeaking(text, conversationActive, turnId);
  }

  function replayTurn(turn: ConversationTurn) {
    if (!turn.assistantText.trim() || !isReadyFlag || isThinking || isSpeaking) return;
    status = 'Replaying response...';
    beginSpeaking(turn.assistantText, conversationActive, turn.id);
  }

  function sendUtterance(rawText: string) {
    const text = rawText.trim();
    if (!text || isThinkingFlag || isSpeakingFlag || !isReadyFlag) return;

    if (isLikelyEcho(text, lastAssistantOutput)) {
      pendingUtterance = '';
      interim = '';
      if (conversationActive) setTimeout(startRecognition, POST_TTS_COOLDOWN_MS);
      return;
    }

    clearPauseTimer();
    pendingUtterance = '';
    interim = '';
    isThinkingFlag = true;
    stopRecognition();
    error = null;
    isThinking = true;
    assistantDraft = '';
    status = 'Bonsai is thinking...';

    const turnId = messageId('turn');
    activeTurnId = turnId;
    const pendingTurn: ConversationTurn = { id: turnId, userText: text, assistantText: '' };
    const nextTurns = [...turnsSnapshot, pendingTurn];
    const workerPayload = buildWorkerMessages(systemPromptSnapshot, turnsToWorkerMessages(nextTurns));
    pendingTurn.llmPrompt = formatWorkerMessagesForDisplay(workerPayload);
    turns = nextTurns;

    bonsaiWorker?.postMessage({ type: 'reset' });
    bonsaiWorker?.postMessage({
      type: 'generate',
      data: workerPayload,
    });
  }

  function updateActiveTurnAssistant(content: string) {
    if (!activeTurnId) return;
    turns = turns.map((turn) =>
      turn.id === activeTurnId ? { ...turn, assistantText: content } : turn,
    );
  }

  function handleBonsai(event: MessageEvent<BonsaiWorkerMessage>) {
    const message = event.data;
    switch (message.status) {
      case 'progress_total':
        status = `Downloading Bonsai ${formatBytes(message.loaded)} / ${formatBytes(message.total)}`;
        break;
      case 'loading':
        status = message.data ?? 'Loading Bonsai...';
        break;
      case 'ready':
        loadedModelId = modelId;
        if (ttsReady) {
          finishLoadingIfReady();
        } else {
          status = `${modelName(modelId)} ready. Waiting for Supertonic...`;
        }
        break;
      case 'start':
        assistantDraft = '';
        break;
      case 'update':
        if (message.tps != null) tokensPerSecond = message.tps;
        if (!message.output) break;
        assistantDraft += message.output;
        updateActiveTurnAssistant(assistantDraft);
        break;
      case 'complete': {
        const output = assistantDraft.trim() || message.output?.trim() || '';
        const completedTurnId = activeTurnId;
        updateActiveTurnAssistant(output);
        activeTurnId = null;
        isThinkingFlag = false;
        isThinking = false;
        status = 'Bonsai answered. Speaking...';
        void persistSession();
        speak(output, completedTurnId);
        break;
      }
      case 'error':
        isLoadingModels = false;
        loadedModelId = null;
        isThinkingFlag = false;
        isThinking = false;
        activeTurnId = null;
        error = message.data ?? 'Bonsai reported an error.';
        status = 'Error.';
        if (conversationActive) startRecognition();
        break;
    }
  }

  function playAudioBlob(audio: ArrayBuffer, mimeType: string) {
    if (!isSpeaking) return;

    const generation = playbackGeneration;
    const blob = new Blob([audio], { type: mimeType || 'audio/wav' });
    const url = URL.createObjectURL(blob);
    const audioNode = new Audio(url);
    audioEl = audioNode;
    audioNode.onended = () => {
      if (generation !== playbackGeneration) {
        URL.revokeObjectURL(url);
        return;
      }
      URL.revokeObjectURL(url);
      const pauseMs = speechSegments[speechSegmentIndex]?.pauseAfterMs ?? 0;
      speechSegmentIndex += 1;
      if (speechSegmentIndex < speechSegments.length) {
        ttsStatus = `Playing part ${speechSegmentIndex} of ${speechSegments.length}...`;
        scheduleNextSegment(pauseMs);
        return;
      }
      ttsStatus = 'Supertonic ready.';
      finishSpeaking();
    };
    audioNode.onerror = () => {
      URL.revokeObjectURL(url);
      if (generation !== playbackGeneration) return;
      error = 'Could not play the Supertonic audio.';
      finishSpeaking();
      if (resumeListeningAfterSpeak && conversationActive) {
        setTimeout(startRecognition, POST_TTS_COOLDOWN_MS);
      }
    };
    ttsStatus =
      speechSegments.length > 1
        ? `Playing part ${speechSegmentIndex + 1} of ${speechSegments.length}...`
        : 'Playing reply.';
    void audioNode.play();
  }

  function handleTts(event: MessageEvent<SupertonicWorkerMessage>) {
    const message = event.data;
    switch (message.status) {
      case 'loading':
        if (isLoadingModels || !ttsReady) {
          ttsStatus = message.data ?? 'Loading Supertonic...';
        }
        break;
      case 'progress':
        if (isLoadingModels || !ttsReady) {
          if (message.file && message.loaded != null && message.total != null) {
            ttsStatus = `Supertonic: ${message.file} (${message.loaded}/${message.total})`;
          }
        }
        break;
      case 'ready':
        ttsReady = true;
        if (isLoadingModels) {
          if (loadedModelId === modelId) {
            finishLoadingIfReady();
          } else {
            ttsStatus = 'Supertonic ready.';
          }
        } else if (!isSpeaking) {
          ttsStatus = 'Supertonic ready.';
        }
        break;
      case 'speaking':
        if (speechSegments.length <= 1) {
          ttsStatus = 'Synthesizing speech...';
        }
        break;
      case 'audio':
        playAudioBlob(message.audio, message.mimeType || 'audio/wav');
        break;
      case 'error':
        if (isLoadingModels) {
          isLoadingModels = false;
          ttsReady = false;
        }
        ttsStatus = 'Supertonic error.';
        error = message.data ?? 'Supertonic reported an error.';
        finishSpeaking();
        if (resumeListeningAfterSpeak && conversationActive) {
          setTimeout(startRecognition, POST_TTS_COOLDOWN_MS);
        }
        break;
    }
  }

  function loadModels() {
    error = null;
    isLoadingModels = true;
    loadedModelId = null;
    ttsReady = false;
    status = `Loading ${selectedModel.name}...`;
    ttsStatus = 'Loading Supertonic...';
    bonsaiWorker?.postMessage({ type: 'load', data: modelId });
    ttsWorker?.postMessage({ type: 'load' });
  }

  function startConversation() {
    if (!speechSupported || !isReady) return;
    conversationActive = true;
    error = null;
    status = 'Listening...';
    startRecognition();
  }

  function stopConversation() {
    conversationActive = false;
    resumeListeningAfterSpeak = false;
    stopSpeaking();
    stopRecognition();
    status = isReady ? `${selectedModel.name} + Supertonic ready.` : 'Load models to begin listening.';
  }

  async function restartConversation() {
    conversationActive = false;
    stopRecognition();
    audioEl?.pause();
    bonsaiWorker?.postMessage({ type: 'reset' });
    turns = [];
    interim = '';
    error = null;
    isThinkingFlag = false;
    isSpeaking = false;
    isSpeakingFlag = false;
    isThinking = false;
    tokensPerSecond = null;
    lastAssistantOutput = '';
    activeTurnId = null;
    speakingTurnId = null;
    playbackGeneration += 1;
    speechSegments = [];
    speechSegmentIndex = 0;
    clearSegmentPauseTimer();
    await clearSession();
    await persistSession();
    status = isReady ? `${selectedModel.name} + Supertonic ready.` : 'Load models to begin listening.';
    ttsStatus = ttsReady ? 'Supertonic ready.' : ttsStatus;
  }

  onMount(() => {
    const storedPrompt = readSystemPrompt();
    if (storedPrompt !== null) {
      systemPrompt = storedPrompt;
    }

    const handleBeforeUnload = () => {
      writeSystemPrompt(systemPrompt);
      flushPersist();
    };
    window.addEventListener('beforeunload', handleBeforeUnload);

    void (async () => {
      await hydrateSession();

      bonsaiWorker = new Worker(BONSAI_WORKER_URL, { type: 'module' });
      ttsWorker = new Worker(SUPERTONIC_WORKER_URL, { type: 'module' });
      bonsaiWorker.addEventListener('message', handleBonsai);
      ttsWorker.addEventListener('message', handleTts);
      bonsaiWorker.postMessage({ type: 'check' });
    })();

    const Recognition = getSpeechRecognition();
    speechSupported = Boolean(Recognition);
    if (Recognition) {
      recognition = new Recognition();
      recognition.continuous = true;
      recognition.interimResults = true;
      recognition.lang = 'en-US';
      recognition.onresult = (event) => {
        if (isThinkingFlag || isSpeakingFlag) return;

        let interimPart = '';
        for (let index = event.resultIndex; index < event.results.length; index += 1) {
          const result = event.results[index];
          const transcript = result[0]?.transcript ?? '';
          if (!transcript) continue;

          if (result.isFinal) {
            pendingUtterance = `${pendingUtterance} ${transcript}`.trim();
            clearPauseTimer();
            pauseTimer = setTimeout(() => {
              sendUtterance(pendingUtterance);
            }, PAUSE_MS);
          } else {
            interimPart += transcript;
          }
        }

        interim = `${pendingUtterance} ${interimPart}`.trim();
      };
      recognition.onerror = (event) => {
        if (event.error === 'no-speech') return;
        recognitionActive = false;
        isListening = false;
        error = event.error ? `Speech recognition: ${event.error}` : 'Speech recognition error.';
      };
      recognition.onstart = () => {
        recognitionActive = true;
        isListening = true;
      };
      recognition.onend = () => {
        recognitionActive = false;
        isListening = false;
        if (conversationActive && !isThinkingFlag && !isSpeakingFlag) {
          setTimeout(startRecognition, POST_TTS_COOLDOWN_MS);
        }
      };
    }

    return () => {
      window.removeEventListener('beforeunload', handleBeforeUnload);
      if (sessionHydrated) flushPersist();
      conversationActive = false;
      if (persistTimer) clearTimeout(persistTimer);
      clearPauseTimer();
      clearSegmentPauseTimer();
      recognitionActive = false;
      recognition?.abort();
      audioEl?.pause();
      bonsaiWorker?.terminate();
      ttsWorker?.terminate();
      bonsaiWorker = null;
      ttsWorker = null;
    };
  });
</script>

<svelte:head>
  <title>Bonsai · Speech to speech</title>
</svelte:head>

<main class="page">
  <div class="container">
    <header class="hero">
      <p class="eyebrow">Speech to speech</p>
      <h1>Talk to Bonsai</h1>
      <p class="lede">Browser-only Bonsai LLM + Supertonic TTS. Conversation is saved locally in IndexedDB.</p>
    </header>

    <section class="panel system-prompt">
      <div class="system-prompt-head">
        <span class="label">System prompt</span>
        <button
          type="button"
          class="btn text"
          onclick={resetSystemPrompt}
          disabled={!sessionHydrated || isThinking || isSpeaking}
        >
          Reset to default
        </button>
      </div>
      <textarea
        class="system-prompt-input"
        value={systemPrompt}
        oninput={handleSystemPromptInput}
        rows="5"
        disabled={!sessionHydrated || isThinking || isSpeaking}
        spellcheck="true"
      ></textarea>
      <p class="system-prompt-hint">
        {#if sessionHydrated}
          Sent as the first message on every turn. Saved automatically on every edit.
        {:else}
          Loading saved session…
        {/if}
      </p>
    </section>

    <section class="panel controls">
      <div class="control-row">
        <label class="field">
          <span class="label">Bonsai model</span>
          <select
            bind:value={modelId}
            disabled={isLoadingModels || isThinking || isSpeaking || isListening}
            onchange={() => {
              loadedModelId = null;
            }}
          >
            {#each MODELS as model (model.id)}
              <option value={model.id}>{model.name} · {model.size}</option>
            {/each}
          </select>
        </label>

        <label class="field">
          <span class="label">Supertonic voice</span>
          <select bind:value={voice} disabled={isSpeaking}>
            {#each VOICES as item (item)}
              <option value={item}>{item}</option>
            {/each}
          </select>
        </label>

        <div class="actions">
          <button
            type="button"
            class="btn primary"
            onclick={loadModels}
            disabled={isLoadingModels || isThinking || isSpeaking || isReady}
          >
            {#if isLoadingModels}
              <span class="spinner" aria-hidden="true"></span>
            {/if}
            {isReady ? 'Loaded' : 'Load'}
          </button>
        </div>
      </div>

      <div class="status-bar">
        <div class="status-main">
          <div class="mic-icon" class:active={isListening} aria-hidden="true">
            {isListening ? '🎤' : '🎙️'}
          </div>
          <div>
            <p class="status-text">{status}</p>
            <p class="status-sub">
              {ttsStatus}{#if tokensPerSecond != null} · {tokensPerSecond.toFixed(1)} tok/s{/if}
            </p>
          </div>
        </div>
        <div class="status-actions">
          {#if conversationActive}
            <button type="button" class="btn outline" onclick={stopConversation}>Stop</button>
          {:else}
            <button
              type="button"
              class="btn primary"
              onclick={startConversation}
              disabled={!speechSupported || !isReady || isThinking || isSpeaking}
            >
              Start
            </button>
          {/if}
        </div>
      </div>

      {#if !speechSupported}
        <p class="alert">Chrome SpeechRecognition is not available in this browser.</p>
      {/if}
      {#if error}
        <p class="alert">{error}</p>
      {/if}
    </section>

    <section class="grid two-col">
      <div class="panel">
        <span class="label">Heard</span>
        <p class="heard">{interim || 'Waiting for speech...'}</p>
      </div>
      <div class="panel">
        <span class="label">Audio output</span>
        <p class="audio-out" class:speaking={isSpeaking}>
          <span aria-hidden="true">{isSpeaking ? '🔊' : '🔈'}</span>
          {isSpeaking ? "Speaking Bonsai's reply..." : 'Supertonic output plays here automatically.'}
        </p>
      </div>
    </section>

    <div class="panel break-speech-bar">
      <label class="field break-field">
        <span class="label">Break speech at</span>
        <select bind:value={speechBreak} disabled={isSpeaking}>
          {#each SPEECH_BREAK_OPTIONS as option (option.value)}
            <option value={option.value}>{option.label}</option>
          {/each}
        </select>
      </label>
      <label class="field break-field">
        <span class="label">TTS language</span>
        <select bind:value={ttsLanguage} disabled={isSpeaking}>
          {#each TTS_LANGUAGE_GROUPS as group (group.label)}
            <optgroup label={group.label}>
              {#each group.languages as language (language.code)}
                <option value={language.code}>{language.label}</option>
              {/each}
            </optgroup>
          {/each}
        </select>
      </label>
    </div>

    <section class="panel conversation">
      <div class="conversation-head">
        <span class="label">Conversation</span>
        <div class="conversation-head-actions">
          <span class="turn-count" aria-hidden="true">{turns.length} turns</span>
          <button
            type="button"
            class="btn outline restart"
            onclick={restartConversation}
            disabled={isThinking || isSpeaking}
          >
            Restart
          </button>
        </div>
      </div>
      <div class="messages">
        {#if turns.length === 0}
          <p class="placeholder">
            Once started, every 500ms pause commits your latest spoken phrase to the full Bonsai conversation.
          </p>
        {:else}
          {#each turns as turn (turn.id)}
            <div class="turn">
              <div class="bubble user">{turn.userText}</div>
              <div class="bubble assistant">
                {#if turn.assistantText || responseTextTab[turn.id]}
                  <div class="response-tabs" role="tablist" aria-label="Response text view">
                    <button
                      type="button"
                      role="tab"
                      class="response-tab"
                      class:active={responseTextTabFor(turn.id) === 'original'}
                      aria-selected={responseTextTabFor(turn.id) === 'original'}
                      onclick={() => setResponseTextTab(turn.id, 'original')}
                    >
                      Original
                    </button>
                    <button
                      type="button"
                      role="tab"
                      class="response-tab"
                      class:active={responseTextTabFor(turn.id) === 'processed'}
                      aria-selected={responseTextTabFor(turn.id) === 'processed'}
                      onclick={() => setResponseTextTab(turn.id, 'processed')}
                    >
                      TTS processed
                    </button>
                    <button
                      type="button"
                      role="tab"
                      class="response-tab"
                      class:active={responseTextTabFor(turn.id) === 'prompt'}
                      aria-selected={responseTextTabFor(turn.id) === 'prompt'}
                      onclick={() => setResponseTextTab(turn.id, 'prompt')}
                    >
                      LLM prompt
                    </button>
                  </div>
                  {#if responseTextTabFor(turn.id) === 'original'}
                    <textarea
                      class="bubble-text response-editor"
                      value={turn.assistantText}
                      oninput={(event) => handleAssistantTextInput(turn.id, event)}
                      disabled={isThinking || isSpeaking}
                      aria-label="Edit original assistant response"
                    ></textarea>
                  {:else}
                    <p class="bubble-text" class:prompt-text={responseTextTabFor(turn.id) === 'prompt'}>
                      {assistantTextForTab(turn)}
                    </p>
                  {/if}
                {:else}
                  <p class="bubble-text">Thinking...</p>
                {/if}
                {#if turn.assistantText}
                  {#if speakingTurnId === turn.id && isSpeaking}
                    <button
                      type="button"
                      class="btn play stop"
                      onclick={stopSpeaking}
                      aria-label="Stop playback"
                    >
                      ■ Stop
                    </button>
                  {:else}
                    <button
                      type="button"
                      class="btn play"
                      onclick={() => replayTurn(turn)}
                      disabled={!isReady || isThinking || isSpeaking}
                      aria-label="Play assistant reply"
                    >
                      ▶ Play
                    </button>
                  {/if}
                {/if}
              </div>
            </div>
          {/each}
        {/if}
      </div>
    </section>
  </div>
</main>

<style>
  .page {
    min-height: 100vh;
    padding: 1.25rem 1rem 3rem;
  }

  .container {
    max-width: 56rem;
    margin: 0 auto;
    display: grid;
    gap: 1.75rem;
  }

  .break-speech-bar {
    display: grid;
    gap: 0.75rem;
    padding: 0.75rem 1rem;
  }

  @media (min-width: 640px) {
    .break-speech-bar {
      grid-template-columns: repeat(2, minmax(0, 1fr));
      align-items: end;
    }
  }

  .break-field {
    min-width: 0;
  }

  .btn.restart {
    flex-shrink: 0;
  }

  .hero {
    text-align: center;
  }

  .eyebrow {
    margin: 0;
    font-size: 0.75rem;
    font-weight: 600;
    letter-spacing: 0.22em;
    text-transform: uppercase;
    color: var(--muted);
  }

  h1 {
    margin: 0.5rem 0 0;
    font-size: clamp(2rem, 5vw, 2.5rem);
    font-weight: 500;
    letter-spacing: -0.04em;
    color: rgba(244, 236, 222, 0.85);
  }

  .lede {
    margin: 0.75rem 0 0;
    color: var(--muted);
    font-size: 0.95rem;
  }

  .panel {
    border: 1px solid var(--border);
    border-radius: 1.25rem;
    background: var(--surface-raised);
    padding: 1rem;
  }

  .controls {
    box-shadow: 0 12px 40px rgba(0, 0, 0, 0.25);
  }

  .system-prompt-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 0.75rem;
    margin-bottom: 0.5rem;
  }

  .system-prompt-input {
    width: 100%;
    min-height: 7rem;
    padding: 0.75rem;
    border: 1px solid var(--border);
    border-radius: 0.75rem;
    background: var(--surface);
    color: var(--cream);
    resize: vertical;
    line-height: 1.5;
    font-family: ui-monospace, 'SF Mono', Menlo, monospace;
    font-size: 0.85rem;
  }

  .system-prompt-input:disabled {
    opacity: 0.65;
  }

  .system-prompt-hint {
    margin: 0.5rem 0 0;
    font-size: 0.75rem;
    color: var(--muted);
  }

  .btn.text {
    min-height: auto;
    padding: 0.25rem 0.5rem;
    background: transparent;
    border: none;
    color: var(--amber);
    font-size: 0.75rem;
    font-weight: 600;
  }

  .btn.text:disabled {
    color: var(--muted);
  }

  .control-row {
    display: grid;
    gap: 0.75rem;
  }

  @media (min-width: 768px) {
    .control-row {
      grid-template-columns: 1fr 1fr auto;
      align-items: end;
    }
  }

  .field {
    display: grid;
    gap: 0.35rem;
  }

  .label {
    font-size: 0.625rem;
    font-weight: 600;
    letter-spacing: 0.2em;
    text-transform: uppercase;
    color: var(--muted);
  }

  select {
    width: 100%;
    height: 2.75rem;
    padding: 0 0.75rem;
    border: 1px solid var(--border);
    background: var(--surface);
  }

  .actions {
    display: flex;
    gap: 0.5rem;
    align-items: end;
  }

  .btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 0.4rem;
    min-height: 2.75rem;
    padding: 0 1rem;
    border: 1px solid transparent;
    font-size: 0.9rem;
    font-weight: 600;
  }

  .btn.primary {
    background: var(--amber);
    color: #1a1208;
  }

  .btn.outline {
    background: transparent;
    border-color: var(--border);
    color: var(--cream);
  }

  .btn.play {
    margin-top: 0.5rem;
    min-height: 2rem;
    padding: 0 0.65rem;
    font-size: 0.75rem;
    background: rgba(232, 165, 94, 0.15);
    border-color: var(--border);
    color: var(--cream);
  }

  .btn.play.stop {
    background: rgba(232, 94, 94, 0.15);
    color: var(--danger);
  }

  .btn.play:disabled {
    opacity: 0.5;
  }

  .spinner {
    width: 1rem;
    height: 1rem;
    border: 2px solid rgba(26, 18, 8, 0.2);
    border-top-color: #1a1208;
    border-radius: 50%;
    animation: spin 0.8s linear infinite;
  }

  @keyframes spin {
    to {
      transform: rotate(360deg);
    }
  }

  .status-bar {
    margin-top: 1rem;
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    justify-content: space-between;
    gap: 0.75rem;
    padding: 0.75rem;
    border: 1px solid var(--border);
    border-radius: 1rem;
    background: rgba(20, 17, 15, 0.55);
  }

  .status-main {
    display: flex;
    align-items: center;
    gap: 0.75rem;
  }

  .mic-icon {
    width: 2.75rem;
    height: 2.75rem;
    display: grid;
    place-items: center;
    border-radius: 1rem;
    background: rgba(232, 165, 94, 0.15);
    font-size: 1.25rem;
  }

  .mic-icon.active {
    background: var(--amber);
  }

  .status-text {
    margin: 0;
    font-size: 0.9rem;
    font-weight: 600;
  }

  .status-sub {
    margin: 0.15rem 0 0;
    font-size: 0.75rem;
    color: var(--muted);
  }

  .alert {
    margin: 0.75rem 0 0;
    padding: 0.65rem 0.75rem;
    border-radius: 0.75rem;
    border: 1px solid rgba(232, 94, 94, 0.25);
    background: var(--danger-soft);
    color: var(--danger);
    font-size: 0.875rem;
  }

  .grid.two-col {
    display: grid;
    gap: 1rem;
  }

  @media (min-width: 768px) {
    .grid.two-col {
      grid-template-columns: 1fr 1fr;
    }
  }

  .heard {
    margin: 0.75rem 0 0;
    min-height: 5rem;
    font-size: 1.25rem;
    line-height: 2rem;
    color: rgba(244, 236, 222, 0.8);
  }

  .audio-out {
    margin: 0.75rem 0 0;
    min-height: 5rem;
    display: flex;
    align-items: center;
    gap: 0.75rem;
    color: var(--muted);
    font-size: 0.9rem;
  }

  .audio-out.speaking {
    color: var(--cream);
    animation: pulse 1.2s ease-in-out infinite;
  }

  @keyframes pulse {
    50% {
      opacity: 0.65;
    }
  }

  .conversation-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 0.75rem;
  }

  .conversation-head-actions {
    display: flex;
    align-items: center;
    gap: 0.75rem;
  }

  .turn-count {
    font-size: 0.75rem;
    color: var(--muted);
  }

  .messages {
    display: grid;
    gap: 1rem;
  }

  .turn {
    display: grid;
    gap: 0.5rem;
  }

  .placeholder {
    margin: 0;
    padding: 1rem;
    border-radius: 0.75rem;
    background: rgba(20, 17, 15, 0.5);
    color: var(--muted);
    font-size: 0.875rem;
  }

  .bubble {
    max-width: 80%;
    padding: 0.75rem 1rem;
    border-radius: 1rem;
    font-size: 0.875rem;
    line-height: 1.5;
  }

  .bubble-text {
    margin: 0;
    white-space: pre-wrap;
  }

  .bubble-text.prompt-text {
    font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, 'Liberation Mono', monospace;
    font-size: 0.8rem;
    line-height: 1.45;
    color: rgba(244, 236, 222, 0.88);
  }

  .response-editor {
    display: block;
    width: 100%;
    box-sizing: border-box;
    min-height: 6rem;
    padding: 0;
    border: 0;
    outline: 0;
    resize: vertical;
    background: transparent;
    color: inherit;
    font: inherit;
  }

  .response-editor:disabled {
    opacity: 0.75;
    cursor: not-allowed;
  }

  .bubble.user {
    margin-left: auto;
    background: var(--amber);
    color: #1a1208;
  }

  .bubble.assistant {
    width: 100%;
    max-width: 100%;
    box-sizing: border-box;
    margin-right: auto;
    border: 1px solid var(--border);
    background: var(--surface);
  }

  .response-tabs {
    display: flex;
    gap: 0.35rem;
    margin-bottom: 0.65rem;
    padding: 0.2rem;
    border-radius: 0.65rem;
    background: rgba(20, 17, 15, 0.45);
  }

  .response-tab {
    flex: 1;
    padding: 0.35rem 0.55rem;
    border: 0;
    border-radius: 0.5rem;
    background: transparent;
    color: var(--muted);
    font: inherit;
    font-size: 0.75rem;
    font-weight: 600;
    cursor: pointer;
    transition:
      background 0.15s ease,
      color 0.15s ease;
  }

  .response-tab:hover {
    color: var(--cream);
  }

  .response-tab.active {
    background: rgba(232, 165, 94, 0.18);
    color: var(--cream);
  }
</style>
