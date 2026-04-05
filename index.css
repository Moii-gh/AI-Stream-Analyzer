import { useState, useEffect, useRef, useCallback } from 'react';
import { GoogleGenAI, Type } from '@google/genai';
import { Camera, Monitor, Mic, Square, Play, Loader2, Download, AlertCircle, Activity, MessageSquare, Code, Lightbulb, FileText, Send, Sparkles, LayoutTemplate } from 'lucide-react';
import { motion, AnimatePresence } from 'motion/react';
import ReactMarkdown from 'react-markdown';

const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });

interface AnalysisResult {
  summary: string;
  keyIdeas: string[];
  codeAnalysis: string | null;
  critique: string | null;
  visualContext: string;
}

interface TranscriptSegment {
  id: string;
  text: string;
  isFinal: boolean;
  timestamp: number;
}

interface ChatMessage {
  id: string;
  role: 'user' | 'assistant';
  text: string;
}

export default function App() {
  const [isRecording, setIsRecording] = useState(false);
  const [sourceType, setSourceType] = useState<'camera' | 'screen'>('screen');
  const [language, setLanguage] = useState('ru-RU');
  const [activeTab, setActiveTab] = useState<'analytics' | 'transcript' | 'chat'>('analytics');
  
  const [transcript, setTranscript] = useState<TranscriptSegment[]>([]);
  const [interimText, setInterimText] = useState('');
  const [currentAnalysis, setCurrentAnalysis] = useState<AnalysisResult | null>(null);
  const [isAnalyzing, setIsAnalyzing] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const [chatMessages, setChatMessages] = useState<ChatMessage[]>([]);
  const [chatInput, setChatInput] = useState('');
  const [isChatting, setIsChatting] = useState(false);

  const videoRef = useRef<HTMLVideoElement>(null);
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const streamRef = useRef<MediaStream | null>(null);
  const recognitionRef = useRef<any>(null);
  const [audioLevel, setAudioLevel] = useState(0);
  const audioContextRef = useRef<AudioContext | null>(null);
  const analyserRef = useRef<AnalyserNode | null>(null);
  const animationFrameRef = useRef<number | null>(null);
  const analysisIntervalRef = useRef<number | null>(null);
  
  const transcriptRef = useRef<TranscriptSegment[]>([]);
  const interimTextRef = useRef<string>('');
  const lastAnalyzedTextRef = useRef<string>('');
  const currentAnalysisRef = useRef<AnalysisResult | null>(null);
  const isRecordingRef = useRef(false);
  
  const transcriptContainerRef = useRef<HTMLDivElement>(null);
  const chatContainerRef = useRef<HTMLDivElement>(null);

  // Синхронизируем стейт с рефами, чтобы коллбеки (например, таймеры) всегда видели актуальные данные
  useEffect(() => { transcriptRef.current = transcript; }, [transcript]);
  useEffect(() => { interimTextRef.current = interimText; }, [interimText]);
  useEffect(() => { currentAnalysisRef.current = currentAnalysis; }, [currentAnalysis]);
  useEffect(() => { isRecordingRef.current = isRecording; }, [isRecording]);

  // Автоскролл для транскрипции, чтобы всегда видеть последние слова
  useEffect(() => {
    if (transcriptContainerRef.current) {
      transcriptContainerRef.current.scrollTop = transcriptContainerRef.current.scrollHeight;
    }
  }, [transcript, interimText, activeTab]);

  useEffect(() => {
    if (chatContainerRef.current) {
      chatContainerRef.current.scrollTop = chatContainerRef.current.scrollHeight;
    }
  }, [chatMessages, isChatting, activeTab]);

  // Главная функция анализа: берет кадр с видео и текст, отправляет в Gemini
  const analyzeCurrentState = useCallback(async () => {
    if (!videoRef.current || !canvasRef.current) return;

    setIsAnalyzing(true);
    try {
      const canvas = canvasRef.current;
      const video = videoRef.current;
      
      // Если видео еще не прогрузилось, просто выходим
      if (video.videoWidth === 0 || video.videoHeight === 0) {
        setIsAnalyzing(false);
        return;
      }

      // Рисуем текущий кадр видео на скрытом канвасе, чтобы получить картинку
      canvas.width = video.videoWidth;
      canvas.height = video.videoHeight;
      const ctx = canvas.getContext('2d');
      if (!ctx) return;
      ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
      
      // Достаем base64 картинки (без префикса data:image/jpeg;base64,)
      const base64Image = canvas.toDataURL('image/jpeg', 0.8).split(',')[1];

      const currentText = transcriptRef.current.map(t => t.text).join(' ') + ' ' + interimTextRef.current;
      const newText = currentText.substring(lastAnalyzedTextRef.current.length).trim();
      
      if (!newText && !base64Image) {
          setIsAnalyzing(false);
          return;
      }

      lastAnalyzedTextRef.current = currentText;

      const prompt = `
        Ты — ИИ-аналитик реального времени. Проанализируй текущий кадр видео и недавнюю транскрипцию аудио.
        Отвечай ИСКЛЮЧИТЕЛЬНО на РУССКОМ ЯЗЫКЕ.
        
        Предыдущий контекст: "${currentAnalysisRef.current?.summary || 'Нет'}"
        Новая транскрипция: "${newText || 'Речь не обнаружена.'}"
        
        Верни JSON со следующей структурой:
        {
          "summary": "Подробное, но структурированное саммари текущего момента трансляции.",
          "keyIdeas": ["Ключевая идея 1", "Ключевая идея 2", "Вывод 1"],
          "codeAnalysis": "Если на экране или в речи есть код: проанализируй его, объясни работу и предложи улучшения (оптимизация, читаемость, архитектура). Если кода нет, верни null.",
          "critique": "Укажи слабые места в текущем материале/выступлении и предложи конкретные улучшения. Если на экране есть интерфейс, предложи концепцию его улучшения (минимализм, modern UI, UX). Если нечего критиковать, верни null.",
          "visualContext": "Краткое описание происходящего на экране."
        }
      `;

      const response = await ai.models.generateContent({
        model: 'gemini-3-flash-preview',
        contents: [
          { text: prompt },
          { inlineData: { data: base64Image, mimeType: 'image/jpeg' } }
        ],
        config: {
          responseMimeType: 'application/json',
          responseSchema: {
            type: Type.OBJECT,
            properties: {
              summary: { type: Type.STRING },
              keyIdeas: { type: Type.ARRAY, items: { type: Type.STRING } },
              codeAnalysis: { type: Type.STRING, nullable: true },
              critique: { type: Type.STRING, nullable: true },
              visualContext: { type: Type.STRING }
            },
            required: ["summary", "keyIdeas", "visualContext"]
          }
        }
      });

      const resultText = response.text;
      if (resultText) {
        const parsed = JSON.parse(resultText) as AnalysisResult;
        setCurrentAnalysis(parsed);
      }

    } catch (err: any) {
      console.error("Analysis error:", err);
      if (err.message?.includes('429') || err.message?.includes('quota')) {
        setError("Превышен лимит запросов к ИИ (Quota exceeded). Анализ приостановлен. Пожалуйста, подождите немного или проверьте ваш API ключ.");
        stopRecording();
      }
    } finally {
      setIsAnalyzing(false);
    }
  }, []);

  const startRecording = async () => {
    try {
      setError(null);
      setTranscript([]);
      setInterimText('');
      setCurrentAnalysis(null);
      lastAnalyzedTextRef.current = '';
      
      let videoStream: MediaStream;
      if (sourceType === 'screen') {
        videoStream = await navigator.mediaDevices.getDisplayMedia({ video: true });
      } else {
        videoStream = await navigator.mediaDevices.getUserMedia({ video: true });
      }
      
      let audioStream: MediaStream | null = null;
      try {
        // Явно запрашиваем доступ к микрофону, иначе транскрипция не заведется
        audioStream = await navigator.mediaDevices.getUserMedia({ audio: true });
      } catch (err) {
        console.warn("Microphone access denied or not available:", err);
        setError("Нет доступа к микрофону. Транскрипция работать не будет. Проверьте разрешения браузера.");
      }
      
      const tracks = [...videoStream.getVideoTracks()];
      if (audioStream) {
        tracks.push(...audioStream.getAudioTracks());
      }
      
      const combinedStream = new MediaStream(tracks);
      streamRef.current = combinedStream;
      
      if (videoRef.current) {
        videoRef.current.srcObject = combinedStream;
      }

      // Настраиваем визуализатор звука (прыгающие зеленые полоски)
      if (audioStream) {
        const audioCtx = new (window.AudioContext || (window as any).webkitAudioContext)();
        audioContextRef.current = audioCtx;
        const analyser = audioCtx.createAnalyser();
        analyser.fftSize = 256;
        analyserRef.current = analyser;
        const source = audioCtx.createMediaStreamSource(audioStream);
        source.connect(analyser);

        const dataArray = new Uint8Array(analyser.frequencyBinCount);
        const updateAudioLevel = () => {
          analyser.getByteFrequencyData(dataArray);
          const sum = dataArray.reduce((a, b) => a + b, 0);
          const average = sum / dataArray.length;
          setAudioLevel(average);
          animationFrameRef.current = requestAnimationFrame(updateAudioLevel);
        };
        updateAudioLevel();
      }

      // Подключаем встроенное в браузер распознавание речи (Web Speech API)
      const SpeechRecognition = (window as any).SpeechRecognition || (window as any).webkitSpeechRecognition;
      if (!SpeechRecognition) {
        setError("Ваш браузер не поддерживает распознавание речи. Пожалуйста, используйте Google Chrome или Edge для работы транскрипции.");
      } else {
        const recognition = new SpeechRecognition();
        recognition.continuous = true;
        recognition.interimResults = true;
        recognition.lang = language;

        recognition.onresult = (event: any) => {
          let currentInterim = '';
          let currentFinal = '';

          for (let i = event.resultIndex; i < event.results.length; ++i) {
            if (event.results[i].isFinal) {
              currentFinal += event.results[i][0].transcript + ' ';
            } else {
              currentInterim += event.results[i][0].transcript;
            }
          }

          if (currentFinal) {
             setTranscript(prev => [...prev, { 
               id: Date.now().toString() + Math.random(), 
               text: currentFinal.trim(), 
               isFinal: true, 
               timestamp: Date.now() 
             }]);
          }
          
          setInterimText(currentInterim);
        };

        recognition.onerror = (event: any) => {
          if (event.error !== 'no-speech') {
            console.error("Speech recognition error", event.error);
          }
          if (event.error === 'not-allowed') {
              setError("Доступ к микрофону запрещен. Транскрипция остановлена.");
          }
        };
        
        // Если распознавание случайно остановилось (например, из-за тишины), перезапускаем его
        recognition.onend = () => {
          if (isRecordingRef.current) {
              try { recognition.start(); } catch(e) {}
          }
        };

        recognition.start();
        recognitionRef.current = recognition;
      }

      setIsRecording(true);
      
      // Делаем первый анализ через 5 секунд после старта, чтобы успел накопиться контекст
      setTimeout(() => {
        if (isRecordingRef.current) analyzeCurrentState();
      }, 5000);
      
      // Запускаем периодический анализ каждые 30 секунд (чтобы не упереться в лимиты API Gemini)
      analysisIntervalRef.current = window.setInterval(() => {
        if (isRecordingRef.current) analyzeCurrentState();
      }, 30000);

    } catch (err: any) {
      console.error(err);
      if (err.name === 'NotFoundError' || err.message?.includes('Requested device not found')) {
        setError(`Не удалось найти ${sourceType === 'camera' ? 'камеру' : 'экран'} для захвата. Проверьте подключенные устройства.`);
      } else if (err.name === 'NotAllowedError' || err.message?.includes('Permission denied')) {
        setError(`Доступ к ${sourceType === 'camera' ? 'камере' : 'экрану'} запрещен.`);
      } else {
        setError(err.message || "Не удалось начать запись");
      }
      stopRecording();
    }
  };

  // Аккуратно всё останавливаем и чистим за собой
  const stopRecording = () => {
    setIsRecording(false);
    
    if (recognitionRef.current) {
      recognitionRef.current.onend = null;
      try { recognitionRef.current.stop(); } catch(e) {}
    }
    
    if (streamRef.current) {
      streamRef.current.getTracks().forEach(track => track.stop());
    }
    
    if (analysisIntervalRef.current) {
      clearInterval(analysisIntervalRef.current);
    }
    if (animationFrameRef.current) {
      cancelAnimationFrame(animationFrameRef.current);
    }
    if (audioContextRef.current) {
      audioContextRef.current.close();
      audioContextRef.current = null;
    }
    setAudioLevel(0);
  };

  // Отправка сообщения в чат с ИИ
  const handleSendMessage = async (e?: React.FormEvent) => {
    e?.preventDefault();
    if (!chatInput.trim() || isChatting) return;
    
    const userText = chatInput.trim();
    const userMsg: ChatMessage = { id: Date.now().toString(), role: 'user', text: userText };
    setChatMessages(prev => [...prev, userMsg]);
    setChatInput('');
    setIsChatting(true);
    
    try {
      const fullTranscript = transcriptRef.current.map(t => t.text).join(' ');
      const context = currentAnalysisRef.current;
      
      const chatPrompt = `
        Ты — интерактивный ИИ-ассистент, помогающий пользователю анализировать текущий стрим/материал.
        Отвечай на вопросы пользователя на основе следующего контекста. Отвечай подробно, структурированно и на русском языке.
        
        ТРАНСКРИПЦИЯ СТРИМА:
        ${fullTranscript || 'Пока нет транскрипции.'}
        
        ТЕКУЩИЙ ВИЗУАЛЬНЫЙ КОНТЕКСТ И АНАЛИЗ:
        ${context ? JSON.stringify(context) : 'Пока нет анализа.'}
        
        ВОПРОС ПОЛЬЗОВАТЕЛЯ:
        ${userText}
      `;
      
      const response = await ai.models.generateContent({
        model: 'gemini-3-flash-preview',
        contents: [{ text: chatPrompt }]
      });
      
      const assistantMsg: ChatMessage = { 
        id: Date.now().toString(), 
        role: 'assistant', 
        text: response.text || 'Не удалось сгенерировать ответ.' 
      };
      setChatMessages(prev => [...prev, assistantMsg]);
    } catch (err) {
      console.error(err);
      setChatMessages(prev => [...prev, { id: Date.now().toString(), role: 'assistant', text: 'Произошла ошибка при обращении к ИИ.' }]);
    } finally {
      setIsChatting(false);
    }
  };

  // Выгрузка всей собранной информации в текстовый файлик
  const exportData = () => {
    const text = transcript.map(t => t.text).join('\n');
    const fullExport = `ТРАНСКРИПЦИЯ:\n${text}\n\nСАММАРИ:\n${currentAnalysis?.summary || ''}\n\nКЛЮЧЕВЫЕ ИДЕИ:\n${currentAnalysis?.keyIdeas?.join('\n') || ''}\n\nАНАЛИЗ КОДА:\n${currentAnalysis?.codeAnalysis || 'Нет'}\n\nКРИТИКА И УЛУЧШЕНИЯ:\n${currentAnalysis?.critique || 'Нет'}`;
    
    const blob = new Blob([fullExport], { type: 'text/plain' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `analysis-${new Date().toISOString()}.txt`;
    a.click();
    URL.revokeObjectURL(url);
  };

  return (
    <div className="min-h-screen bg-[#0a0a0a] text-neutral-200 font-sans selection:bg-white/20">
      {/* Шапка приложения */}
      <header className="sticky top-0 z-20 bg-[#0a0a0a]/80 backdrop-blur-2xl border-b border-white/5">
        <div className="max-w-7xl mx-auto px-6 h-20 flex items-center justify-between">
          <div className="flex items-center gap-4">
            <div className="w-10 h-10 rounded-2xl bg-white flex items-center justify-center text-black shadow-lg shadow-white/10">
              <Sparkles className="w-5 h-5" />
            </div>
            <div>
              <h1 className="text-lg font-semibold tracking-tight text-white">AI Stream Analyzer</h1>
              <p className="text-xs text-neutral-400 font-medium">Real-time intelligence</p>
            </div>
          </div>
          
          <div className="flex items-center gap-4">
            <div className="flex bg-neutral-900/50 p-1.5 rounded-full border border-white/5">
              <button
                onClick={() => setSourceType('camera')}
                disabled={isRecording}
                className={`px-4 py-2 rounded-full text-sm font-medium flex items-center gap-2 transition-all ${sourceType === 'camera' ? 'bg-white text-black shadow-sm' : 'text-neutral-400 hover:text-white'} disabled:opacity-50`}
              >
                <Camera className="w-4 h-4" /> Камера
              </button>
              <button
                onClick={() => setSourceType('screen')}
                disabled={isRecording}
                className={`px-4 py-2 rounded-full text-sm font-medium flex items-center gap-2 transition-all ${sourceType === 'screen' ? 'bg-white text-black shadow-sm' : 'text-neutral-400 hover:text-white'} disabled:opacity-50`}
              >
                <Monitor className="w-4 h-4" /> Экран
              </button>
            </div>

            <motion.button
              whileHover={{ scale: 1.03 }}
              whileTap={{ scale: 0.97 }}
              onClick={isRecording ? stopRecording : startRecording}
              className={`px-6 py-2.5 rounded-full text-sm font-semibold flex items-center gap-2 transition-colors shadow-lg ${
                isRecording 
                  ? 'bg-red-500/10 text-red-500 hover:bg-red-500/20 border border-red-500/20' 
                  : 'bg-white text-black hover:bg-neutral-200'
              }`}
            >
              {isRecording ? (
                <><Square className="w-4 h-4 fill-current" /> Остановить</>
              ) : (
                <><Play className="w-4 h-4 fill-current" /> Начать анализ</>
              )}
            </motion.button>
          </div>
        </div>
      </header>

      {/* Основной контент */}
      <main className="max-w-7xl mx-auto px-6 py-8 grid grid-cols-1 lg:grid-cols-12 gap-8">
        
        {error && (
          <div className="lg:col-span-12 bg-red-500/10 border border-red-500/20 text-red-400 px-5 py-4 rounded-3xl flex items-center gap-3">
            <AlertCircle className="w-5 h-5 shrink-0" />
            <p className="text-sm font-medium">{error}</p>
          </div>
        )}

        {/* Левая колонка: Видеопоток и визуальный контекст */}
        <div className="lg:col-span-5 flex flex-col gap-6">
          <div className="rounded-3xl overflow-hidden bg-neutral-900/50 border border-white/5 aspect-video relative group shadow-2xl">
            <video 
              ref={videoRef} 
              autoPlay 
              playsInline 
              muted 
              className="w-full h-full object-cover"
            />
            {!isRecording && (
              <div className="absolute inset-0 flex flex-col items-center justify-center text-neutral-500 bg-neutral-950/80 backdrop-blur-sm">
                {sourceType === 'camera' ? <Camera className="w-10 h-10 mb-4 opacity-50" /> : <Monitor className="w-10 h-10 mb-4 opacity-50" />}
                <p className="text-sm font-medium">Ожидание захвата</p>
              </div>
            )}
            <AnimatePresence>
              {isRecording && (
                <motion.div 
                  initial={{ opacity: 0, scale: 0.8 }}
                  animate={{ opacity: 1, scale: 1 }}
                  exit={{ opacity: 0, scale: 0.8 }}
                  className="absolute top-4 left-4 flex items-center gap-2 bg-black/60 backdrop-blur-xl px-4 py-2 rounded-full border border-white/10"
                >
                  <motion.div 
                    animate={{ opacity: [1, 0.4, 1] }} 
                    transition={{ repeat: Infinity, duration: 1.5, ease: "easeInOut" }}
                    className="w-2 h-2 rounded-full bg-red-500 shadow-[0_0_8px_rgba(239,68,68,0.8)]" 
                  />
                  <span className="text-xs font-bold text-white tracking-widest uppercase">Live</span>
                </motion.div>
              )}
            </AnimatePresence>
          </div>

          {/* Панель с описанием того, что ИИ видит на экране */}
          <div className="bg-neutral-900/40 border border-white/5 rounded-3xl p-6 shadow-xl backdrop-blur-xl flex-1">
            <div className="flex items-center gap-3 mb-4">
              <div className="w-8 h-8 rounded-full bg-neutral-800 flex items-center justify-center text-neutral-400">
                <Camera className="w-4 h-4" />
              </div>
              <h3 className="text-sm font-semibold text-white">Визуальный контекст</h3>
              {isAnalyzing && <Loader2 className="w-4 h-4 animate-spin ml-auto text-neutral-400" />}
            </div>
            <AnimatePresence mode="wait">
              <motion.p 
                key={currentAnalysis?.visualContext || 'empty'}
                initial={{ opacity: 0 }}
                animate={{ opacity: 1 }}
                exit={{ opacity: 0 }}
                transition={{ duration: 0.3 }}
                className="text-sm text-neutral-400 leading-relaxed"
              >
                {currentAnalysis?.visualContext || (isRecording ? "Анализируем видеопоток..." : "Здесь появится описание происходящего на экране.")}
              </motion.p>
            </AnimatePresence>
          </div>
        </div>

        {/* Правая колонка: Вкладки с аналитикой, транскрипцией и чатом */}
        <div className="lg:col-span-7 flex flex-col h-[calc(100vh-10rem)]">
          
          {/* Переключатели вкладок */}
          <div className="flex items-center gap-2 bg-neutral-900/50 p-1.5 rounded-full border border-white/5 mb-6 shrink-0 w-fit">
            <button
              onClick={() => setActiveTab('analytics')}
              className={`px-5 py-2 rounded-full text-sm font-medium flex items-center gap-2 transition-all ${activeTab === 'analytics' ? 'bg-white text-black shadow-sm' : 'text-neutral-400 hover:text-white'}`}
            >
              <Activity className="w-4 h-4" /> Аналитика
            </button>
            <button
              onClick={() => setActiveTab('transcript')}
              className={`px-5 py-2 rounded-full text-sm font-medium flex items-center gap-2 transition-all ${activeTab === 'transcript' ? 'bg-white text-black shadow-sm' : 'text-neutral-400 hover:text-white'}`}
            >
              <FileText className="w-4 h-4" /> Транскрипция
            </button>
            <button
              onClick={() => setActiveTab('chat')}
              className={`px-5 py-2 rounded-full text-sm font-medium flex items-center gap-2 transition-all ${activeTab === 'chat' ? 'bg-white text-black shadow-sm' : 'text-neutral-400 hover:text-white'}`}
            >
              <MessageSquare className="w-4 h-4" /> Чат с ИИ
            </button>
          </div>

          {/* Содержимое выбранной вкладки */}
          <div className="bg-neutral-900/40 border border-white/5 rounded-3xl p-6 shadow-xl backdrop-blur-xl flex-1 flex flex-col min-h-0 overflow-hidden relative">
            <AnimatePresence mode="wait">
            {/* Вкладка: Аналитика */}
            {activeTab === 'analytics' && (
              <motion.div 
                key="analytics"
                initial={{ opacity: 0, y: 10, filter: 'blur(4px)' }}
                animate={{ opacity: 1, y: 0, filter: 'blur(0px)' }}
                exit={{ opacity: 0, y: -10, filter: 'blur(4px)' }}
                transition={{ duration: 0.3, ease: "easeOut" }}
                className="flex-1 overflow-y-auto pr-4 space-y-8 custom-scrollbar"
              >
                <div className="flex items-center justify-between">
                  <h2 className="text-xl font-semibold text-white">Сводка в реальном времени</h2>
                  <button 
                    onClick={exportData}
                    disabled={transcript.length === 0 && !currentAnalysis}
                    className="text-xs font-medium flex items-center gap-2 hover:bg-neutral-800 disabled:opacity-50 transition-colors bg-neutral-900 border border-white/5 px-4 py-2 rounded-full"
                  >
                    <Download className="w-3 h-3" /> Экспорт
                  </button>
                </div>

                <div className="space-y-3">
                  <div className="flex items-center gap-2 text-neutral-400">
                    <Lightbulb className="w-4 h-4" />
                    <h3 className="text-sm font-medium uppercase tracking-wider">Главное</h3>
                  </div>
                  <p className="text-sm text-neutral-300 leading-relaxed">
                    {currentAnalysis?.summary || (isRecording ? "Генерация саммари..." : "Саммари появится после начала анализа.")}
                  </p>
                </div>

                <div className="space-y-3">
                  <div className="flex items-center gap-2 text-neutral-400">
                    <Activity className="w-4 h-4" />
                    <h3 className="text-sm font-medium uppercase tracking-wider">Ключевые идеи</h3>
                  </div>
                  <ul className="space-y-3">
                    {currentAnalysis?.keyIdeas ? (
                      currentAnalysis.keyIdeas.map((idea, i) => (
                        <motion.li 
                          initial={{ opacity: 0, x: -15, filter: 'blur(4px)' }} 
                          animate={{ opacity: 1, x: 0, filter: 'blur(0px)' }}
                          transition={{ delay: i * 0.1, type: "spring", stiffness: 400, damping: 30 }}
                          key={i} className="text-sm text-neutral-300 flex items-start gap-3 bg-neutral-800/30 p-3 rounded-2xl"
                        >
                          <span className="text-white mt-0.5">•</span>
                          <span>{idea}</span>
                        </motion.li>
                      ))
                    ) : (
                      <li className="text-sm text-neutral-500">{isRecording ? "Извлечение идей..." : "Пока нет идей..."}</li>
                    )}
                  </ul>
                </div>

                {currentAnalysis?.codeAnalysis && (
                  <div className="space-y-3">
                    <div className="flex items-center gap-2 text-neutral-400">
                      <Code className="w-4 h-4" />
                      <h3 className="text-sm font-medium uppercase tracking-wider">Анализ кода</h3>
                    </div>
                    <div className="text-sm text-neutral-300 leading-relaxed bg-neutral-800/30 p-4 rounded-2xl border border-white/5">
                      <div className="markdown-body prose prose-invert prose-sm max-w-none">
                        <ReactMarkdown>{currentAnalysis.codeAnalysis}</ReactMarkdown>
                      </div>
                    </div>
                  </div>
                )}

                {currentAnalysis?.critique && (
                  <div className="space-y-3">
                    <div className="flex items-center gap-2 text-neutral-400">
                      <LayoutTemplate className="w-4 h-4" />
                      <h3 className="text-sm font-medium uppercase tracking-wider">Критика и UX/UI</h3>
                    </div>
                    <div className="text-sm text-neutral-300 leading-relaxed bg-neutral-800/30 p-4 rounded-2xl border border-white/5">
                      <div className="markdown-body prose prose-invert prose-sm max-w-none">
                        <ReactMarkdown>{currentAnalysis.critique}</ReactMarkdown>
                      </div>
                    </div>
                  </div>
                )}
              </motion.div>
            )}

            {/* Вкладка: Транскрипция */}
            {activeTab === 'transcript' && (
              <motion.div 
                key="transcript"
                initial={{ opacity: 0, y: 10, filter: 'blur(4px)' }}
                animate={{ opacity: 1, y: 0, filter: 'blur(0px)' }}
                exit={{ opacity: 0, y: -10, filter: 'blur(4px)' }}
                transition={{ duration: 0.3, ease: "easeOut" }}
                className="flex-1 flex flex-col h-full"
              >
                <div className="flex items-center gap-2 text-neutral-400 mb-6 shrink-0">
                  <Mic className="w-4 h-4" />
                  <h3 className="text-sm font-medium uppercase tracking-wider">Прямая транскрипция</h3>
                  {isRecording && (
                    <div className="ml-auto flex items-center gap-2">
                      <span className="text-xs text-neutral-500">Микрофон</span>
                      <div className="flex items-center gap-0.5 h-4">
                        {[...Array(5)].map((_, i) => (
                          <motion.div
                            key={i}
                            animate={{ height: Math.max(4, (audioLevel / 255) * 16 * (Math.random() * 0.5 + 0.5)) }}
                            className="w-1 bg-green-500 rounded-full"
                            style={{ minHeight: '4px' }}
                          />
                        ))}
                      </div>
                    </div>
                  )}
                </div>
                <div 
                  className="flex-1 overflow-y-auto space-y-4 pr-4 custom-scrollbar" 
                  ref={transcriptContainerRef}
                >
                  {transcript.map((seg) => (
                    <motion.div 
                      initial={{ opacity: 0, x: -10, scale: 0.98 }} 
                      animate={{ opacity: 1, x: 0, scale: 1 }}
                      transition={{ type: "spring", stiffness: 500, damping: 30 }}
                      key={seg.id} 
                      className="text-sm text-neutral-300 leading-relaxed bg-neutral-800/20 p-3 rounded-2xl"
                    >
                      {seg.text}
                    </motion.div>
                  ))}
                  {interimText && (
                    <div className="text-sm text-neutral-500 leading-relaxed italic p-3">
                      {interimText}
                    </div>
                  )}
                  {transcript.length === 0 && !interimText && (
                    <div className="text-sm text-neutral-600 h-full flex items-center justify-center">
                      {isRecording ? "Слушаю..." : "Ожидание речи..."}
                    </div>
                  )}
                </div>
              </motion.div>
            )}

            {/* Вкладка: Чат с ИИ */}
            {activeTab === 'chat' && (
              <motion.div 
                key="chat"
                initial={{ opacity: 0, y: 10, filter: 'blur(4px)' }}
                animate={{ opacity: 1, y: 0, filter: 'blur(0px)' }}
                exit={{ opacity: 0, y: -10, filter: 'blur(4px)' }}
                transition={{ duration: 0.3, ease: "easeOut" }}
                className="flex-1 flex flex-col h-full"
              >
                <div className="flex items-center gap-2 text-neutral-400 mb-6 shrink-0">
                  <MessageSquare className="w-4 h-4" />
                  <h3 className="text-sm font-medium uppercase tracking-wider">Интерактивный ассистент</h3>
                </div>
                
                <div className="flex-1 overflow-y-auto space-y-6 pr-4 custom-scrollbar mb-4" ref={chatContainerRef}>
                  {chatMessages.length === 0 ? (
                    <div className="h-full flex flex-col items-center justify-center text-center space-y-4 text-neutral-500">
                      <div className="w-12 h-12 rounded-full bg-neutral-800/50 flex items-center justify-center">
                        <Sparkles className="w-6 h-6 text-neutral-400" />
                      </div>
                      <p className="text-sm max-w-xs">
                        Задайте вопрос по содержанию стрима, коду или дизайну. ИИ ответит на основе контекста.
                      </p>
                    </div>
                  ) : (
                    chatMessages.map((msg) => (
                      <motion.div 
                        initial={{ opacity: 0, y: 10, scale: 0.95, transformOrigin: msg.role === 'user' ? 'bottom right' : 'bottom left' }}
                        animate={{ opacity: 1, y: 0, scale: 1 }}
                        transition={{ type: "spring", stiffness: 400, damping: 25 }}
                        key={msg.id} 
                        className={`flex ${msg.role === 'user' ? 'justify-end' : 'justify-start'}`}
                      >
                        <div className={`max-w-[85%] p-4 text-sm leading-relaxed ${
                          msg.role === 'user' 
                            ? 'bg-white text-black rounded-3xl rounded-tr-sm' 
                            : 'bg-neutral-800/50 text-neutral-200 border border-white/5 rounded-3xl rounded-tl-sm'
                        }`}>
                          {msg.role === 'user' ? (
                            msg.text
                          ) : (
                            <div className="markdown-body prose prose-invert prose-sm max-w-none">
                              <ReactMarkdown>{msg.text}</ReactMarkdown>
                            </div>
                          )}
                        </div>
                      </motion.div>
                    ))
                  )}
                  {isChatting && (
                    <div className="flex justify-start">
                      <div className="bg-neutral-800/50 border border-white/5 rounded-3xl rounded-tl-sm p-4 flex items-center gap-2">
                        <div className="w-1.5 h-1.5 bg-neutral-400 rounded-full animate-bounce" />
                        <div className="w-1.5 h-1.5 bg-neutral-400 rounded-full animate-bounce" style={{ animationDelay: '0.2s' }} />
                        <div className="w-1.5 h-1.5 bg-neutral-400 rounded-full animate-bounce" style={{ animationDelay: '0.4s' }} />
                      </div>
                    </div>
                  )}
                </div>

                <form onSubmit={handleSendMessage} className="shrink-0 relative">
                  <input
                    type="text"
                    value={chatInput}
                    onChange={(e) => setChatInput(e.target.value)}
                    placeholder="Спросите что-нибудь о стриме..."
                    className="w-full bg-neutral-900 border border-white/10 rounded-full pl-6 pr-14 py-4 text-sm text-white placeholder-neutral-500 focus:outline-none focus:border-white/20 focus:ring-1 focus:ring-white/20 transition-all"
                    disabled={isChatting}
                  />
                  <motion.button
                    whileHover={{ scale: 1.05 }}
                    whileTap={{ scale: 0.95 }}
                    type="submit"
                    disabled={!chatInput.trim() || isChatting}
                    className="absolute right-2 top-1/2 -translate-y-1/2 w-10 h-10 rounded-full bg-white text-black flex items-center justify-center disabled:opacity-50 disabled:bg-neutral-800 disabled:text-neutral-500 transition-colors"
                  >
                    <Send className="w-4 h-4 ml-0.5" />
                  </motion.button>
                </form>
              </motion.div>
            )}
            </AnimatePresence>

          </div>
        </div>
      </main>
      
      <canvas ref={canvasRef} className="hidden" />
      
      <style>{`
        .custom-scrollbar::-webkit-scrollbar { width: 6px; }
        .custom-scrollbar::-webkit-scrollbar-track { background: transparent; }
        .custom-scrollbar::-webkit-scrollbar-thumb { background: #333; border-radius: 10px; }
        .custom-scrollbar::-webkit-scrollbar-thumb:hover { background: #555; }
      `}</style>
    </div>
  );
}
