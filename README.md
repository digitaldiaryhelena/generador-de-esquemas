# generador-de-esquemas
import React, { useState, useRef } from 'react';
import { 
  Upload, FileText, Image as ImageIcon, Trash2, Sparkles, Download, Copy, Check, 
  RefreshCw, ZoomIn, ZoomOut, Layers, BookOpen, BrainCircuit, 
  HelpCircle, ChevronRight, ChevronDown, Settings, AlertCircle, ArrowLeft,
  X, Search, Award, CheckCircle2, XCircle, Printer, Heart, Flower2
} from 'lucide-react';

export default function App() {
  // State for uploaded files
  const [files, setFiles] = useState([]);
  const [isDragging, setIsDragging] = useState(false);
  
  // Customization Options
  const [outputStyle, setOutputStyle] = useState('mindmap'); // mindmap, outline, flashcards, summary
  const [depthLevel, setDepthLevel] = useState('exhaustivo'); // ejecutivo, detallado, exhaustivo
  const [customFocus, setCustomFocus] = useState('');
  
  // Processing state
  const [isProcessing, setIsProcessing] = useState(false);
  const [processingProgress, setProcessingProgress] = useState(0);
  const [processingStatus, setProcessingStatus] = useState('');
  const [errorMessage, setErrorMessage] = useState(null);

  // Result state
  const [resultData, setResultData] = useState(null);
  const [activeTab, setActiveTab] = useState('mindmap'); // mindmap, outline, flashcards, quiz
  
  // Interactive Mind Map state
  const [zoomLevel, setZoomLevel] = useState(1);
  const [selectedNode, setSelectedNode] = useState(null);
  const [searchQuery, setSearchQuery] = useState('');
  const [collapsedNodes, setCollapsedNodes] = useState({});

  // Quiz state
  const [quizAnswers, setQuizAnswers] = useState({});
  const [quizSubmitted, setQuizSubmitted] = useState(false);
  const [quizScore, setQuizScore] = useState(0);

  // Flashcards state
  const [currentCardIndex, setCurrentCardIndex] = useState(0);
  const [isCardFlipped, setIsCardFlipped] = useState(false);

  // Copy notification state
  const [copied, setCopied] = useState(false);

  // Maximum files limit
  const MAX_FILES = 100;

  // File Upload Handlers
  const handleFileChange = (e) => {
    const selectedFiles = Array.from(e.target.files);
    addFiles(selectedFiles);
  };

  const addFiles = (newFiles) => {
    const imageFiles = newFiles.filter(file => file.type.startsWith('image/'));
    
    if (imageFiles.length + files.length > MAX_FILES) {
      alert(`Solo puedes subir un máximo de ${MAX_FILES} imágenes a la vez.`);
      const allowed = imageFiles.slice(0, MAX_FILES - files.length);
      processFilesForPreview(allowed);
    } else {
      processFilesForPreview(imageFiles);
    }
  };

  const processFilesForPreview = (validFiles) => {
    const filePromises = validFiles.map(file => {
      return new Promise((resolve) => {
        const reader = new FileReader();
        reader.onload = (e) => {
          resolve({
            id: Math.random().toString(36).substring(2, 9),
            file: file,
            name: file.name,
            size: (file.size / (1024 * 1024)).toFixed(2) + ' MB',
            preview: e.target.result,
            base64: e.target.result.split(',')[1],
            mimeType: file.type
          });
        };
        reader.readAsDataURL(file);
      });
    });

    Promise.all(filePromises).then(newFileObjs => {
      setFiles(prev => [...prev, ...newFileObjs]);
    });
  };

  const removeFile = (id) => {
    setFiles(prev => prev.filter(f => f.id !== id));
  };

  const clearAllFiles = () => {
    setFiles([]);
  };

  const handleDragOver = (e) => {
    e.preventDefault();
    setIsDragging(true);
  };

  const handleDragLeave = () => {
    setIsDragging(false);
  };

  const handleDrop = (e) => {
    e.preventDefault();
    setIsDragging(false);
    if (e.dataTransfer.files && e.dataTransfer.files.length > 0) {
      addFiles(Array.from(e.dataTransfer.files));
    }
  };

  // Helper function to call Gemini API
  const generateOutlineWithGemini = async () => {
    if (files.length === 0) return;

    setIsProcessing(true);
    setErrorMessage(null);
    setProcessingProgress(10);
    setProcessingStatus('Preparando y procesando imágenes de apuntes...');

    try {
      await new Promise(r => setTimeout(r, 600));
      setProcessingProgress(35);
      setProcessingStatus(`Analizando ${files.length} archivo(s) con la IA...`);

      const depthPrompts = {
        ejecutivo: "Crea un esquema sintético y elegante, ideal para repaso rápido.",
        detallado: "Crea un esquema altamente estructurado y equilibrado con definiciones clave.",
        exhaustivo: "Crea un esquema SÚPER COMPLETO, ultra exhaustivo con todos los conceptos, clasificaciones, fórmulas y detalles presentes en los apuntes."
      };

      const systemPrompt = `Eres una tutora académica experta de estilo elegante y súper organizada. Tu objetivo es sintetizar todo el contenido de las fotos de apuntes en una guía de estudio de alto rendimiento.
Responde ÚNICAMENTE con un JSON válido estructurado con el siguiente formato exacto sin texto adicional:

{
  "title": "Título Principal del Tema",
  "summary": "Resumen ejecutivo del tema estudiado.",
  "rootNode": {
    "id": "root",
    "label": "Concepto Central",
    "type": "main",
    "description": "Síntesis del tema central",
    "children": [
      {
        "id": "node-1",
        "label": "Subtema Principal 1",
        "description": "Explicación académica clara",
        "details": ["Punto clave A", "Definición o término imprescindible"],
        "children": [
          {
            "id": "node-1-1",
            "label": "Detalle Específico 1.1",
            "description": "Descripción conceptual",
            "details": ["Ejemplo relevante"],
            "children": []
          }
        ]
      }
    ]
  },
  "flashcards": [
    {
      "id": "fc-1",
      "question": "¿Pregunta clave para estudiar?",
      "answer": "Respuesta precisa y directa.",
      "category": "Sección"
    }
  ],
  "quiz": [
    {
      "id": "q-1",
      "question": "¿Pregunta conceptual tipo examen?",
      "options": ["Opción A", "Opción B", "Opción C", "Opción D"],
      "correctIndex": 0,
      "explanation": "Fundamento teórico de la respuesta correcta."
    }
  ]
}

Nivel solicitado: ${depthPrompts[depthLevel]}
${customFocus ? `Instrucciones particulares: ${customFocus}` : ''}`;

      const parts = [
        { text: systemPrompt + "\n\nExamina minuciosamente las imágenes adjuntas y genera el JSON solicitado." }
      ];

      files.forEach((fileObj) => {
        parts.push({
          inlineData: {
            mimeType: fileObj.mimeType || 'image/jpeg',
            data: fileObj.base64
          }
        });
      });

      setProcessingProgress(65);
      setProcessingStatus('Estructurando esquema, fichas y preguntas...');

      const apiKey = ""; 
      const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=${apiKey}`;

      const response = await fetch(apiUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          contents: [{ parts }],
          generationConfig: {
            responseMimeType: "application/json",
            temperature: 0.2
          }
        })
      });

      setProcessingProgress(85);
      setProcessingStatus('Refinando la presentación en estilo Rosa Francés...');

      if (!response.ok) {
        throw new Error(`Error en la llamada a la API (${response.status})`);
      }

      const result = await response.json();
      const rawText = result.candidates?.[0]?.content?.parts?.[0]?.text;

      if (!rawText) {
        throw new Error('No se recibió una respuesta válida de la IA.');
      }

      let parsedJson;
      try {
        parsedJson = JSON.parse(rawText);
      } catch (parseErr) {
        const cleanedText = rawText.replace(/```json/g, '').replace(/```/g, '').trim();
        parsedJson = JSON.parse(cleanedText);
      }

      setProcessingProgress(100);
      setResultData(parsedJson);
      setSelectedNode(parsedJson.rootNode);
      setActiveTab(outputStyle === 'flashcards' ? 'flashcards' : 'mindmap');

    } catch (err) {
      console.error('Error generating esquema:', err);
      setErrorMessage(err.message || 'Error al procesar los documentos. Inténtalo de nuevo.');
    } finally {
      setIsProcessing(false);
    }
  };

  const toggleNodeCollapse = (nodeId) => {
    setCollapsedNodes(prev => ({
      ...prev,
      [nodeId]: !prev[nodeId]
    }));
  };

  const copyAsText = () => {
    if (!resultData) return;
    
    let formattedText = `# ${resultData.title}\n\n${resultData.summary}\n\n`;

    const formatNode = (node, depth = 0) => {
      const indent = '  '.repeat(depth);
      let text = `${indent}- **${node.label}**: ${node.description || ''}\n`;
      if (node.details && node.details.length > 0) {
        node.details.forEach(d => {
          text += `${indent}  * ${d}\n`;
        });
      }
      if (node.children && node.children.length > 0) {
        node.children.forEach(child => {
          text += formatNode(child, depth + 1);
        });
      }
      return text;
    };

    formattedText += formatNode(resultData.rootNode);

    navigator.clipboard.writeText(formattedText);
    setCopied(true);
    setTimeout(() => setCopied(false), 2000);
  };

  const exportPDF = () => {
    window.print();
  };

  const exportMarkdown = () => {
    if (!resultData) return;
    
    let formattedText = `# ${resultData.title}\n\n> ${resultData.summary}\n\n## Esquema de Contenidos\n\n`;

    const formatNode = (node, depth = 1) => {
      const headingPrefix = '#'.repeat(Math.min(depth + 1, 6));
      let text = `${headingPrefix} ${node.label}\n${node.description ? node.description + '\n\n' : ''}`;
      
      if (node.details && node.details.length > 0) {
        node.details.forEach(d => {
          text += `- ${d}\n`;
        });
        text += '\n';
      }
      if (node.children && node.children.length > 0) {
        node.children.forEach(child => {
          text += formatNode(child, depth + 1);
        });
      }
      return text;
    };

    formattedText += formatNode(resultData.rootNode);

    const blob = new Blob([formattedText], { type: 'text/markdown' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `${resultData.title.toLowerCase().replace(/\s+/g, '_')}_esquema.md`;
    a.click();
    URL.revokeObjectURL(url);
  };

  const handleQuizOptionSelect = (qId, optionIdx) => {
    if (quizSubmitted) return;
    setQuizAnswers(prev => ({
      ...prev,
      [qId]: optionIdx
    }));
  };

  const submitQuiz = () => {
    if (!resultData || !resultData.quiz) return;
    let score = 0;
    resultData.quiz.forEach(q => {
      if (quizAnswers[q.id] === q.correctIndex) {
        score += 1;
      }
    });
    setQuizScore(score);
    setQuizSubmitted(true);
  };

  const resetQuiz = () => {
    setQuizAnswers({});
    setQuizSubmitted(false);
    setQuizScore(0);
  };

  return (
    <div className="min-h-screen bg-[#FAF0F3] text-[#4A1829] flex flex-col font-sans selection:bg-[#E66792] selection:text-white">
      
      {/* Custom Styles for French Rose Palette & PDF Printing */}
      <style>{`
        :root {
          --french-rose-light: #FAF0F3;
          --french-rose-card: #F5E3E9;
          --french-rose-border: #E8C2CE;
          --french-rose: #E66792;
          --french-rose-deep: #D4507D;
          --french-rose-dark: #9E2B52;
          --french-rose-wine: #731A38;
        }

        @media print {
          body * {
            visibility: hidden;
          }
          #printable-pdf-area, #printable-pdf-area * {
            visibility: visible;
          }
          #printable-pdf-area {
            position: absolute;
            left: 0;
            top: 0;
            width: 100%;
            padding: 24px;
            color: #2D0D17;
            background: #FFFFFF;
          }
          .no-print {
            display: none !important;
          }
        }
      `}</style>

      {/* French Rose Header */}
      <header className="no-print border-b border-[#E8C2CE] bg-[#FAF0F3]/90 backdrop-blur-md sticky top-0 z-50 px-4 lg:px-8 py-3.5 flex items-center justify-between shadow-sm">
        <div className="flex items-center gap-3">
          <div className="p-2 bg-gradient-to-tr from-[#9E2B52] via-[#D4507D] to-[#E66792] rounded-2xl shadow-md shadow-[#E66792]/30">
            <Flower2 className="w-5 h-5 text-white" />
          </div>
          <div>
            <h1 className="text-xl font-bold tracking-tight bg-gradient-to-r from-[#731A38] via-[#9E2B52] to-[#D4507D] bg-clip-text text-transparent">
              EsquemaAI <span className="text-[11px] font-semibold text-[#9E2B52] bg-[#F5E3E9] px-2.5 py-0.5 rounded-full border border-[#E8C2CE] ml-1">Rose Français</span>
            </h1>
            <p className="text-xs text-[#9E2B52]/80">Generador de Esquemas Académicos y PDFs de Estudio</p>
          </div>
        </div>

        {resultData && (
          <div className="flex items-center gap-2">
            <button
              onClick={() => setResultData(null)}
              className="flex items-center gap-2 px-3.5 py-1.5 text-xs font-semibold bg-white hover:bg-[#F5E3E9] text-[#731A38] rounded-xl border border-[#E8C2CE] transition shadow-sm"
            >
              <ArrowLeft className="w-3.5 h-3.5" />
              Nuevo Esquema
            </button>
          </div>
        )}
      </header>

      {/* Main Content Area */}
      <main className="flex-1 flex flex-col overflow-hidden">
        {!resultData ? (
          /* DASHBOARD INPUT VIEW */
          <div className="max-w-7xl w-full mx-auto p-4 sm:p-6 lg:p-8 flex flex-col lg:flex-row gap-8 flex-1">
            
            {/* Left Column: Upload & Settings */}
            <div className="w-full lg:w-5/12 flex flex-col gap-6">
              
              {/* File Upload Zone */}
              <div className="bg-white/80 border border-[#E8C2CE] rounded-3xl p-5 shadow-lg shadow-[#E8C2CE]/20 backdrop-blur-sm flex flex-col">
                <div className="flex items-center justify-between mb-4">
                  <div className="flex items-center gap-2">
                    <ImageIcon className="w-4 h-4 text-[#D4507D]" />
                    <h2 className="font-semibold text-[#731A38] text-sm">Fotos de Apuntes</h2>
                  </div>
                  <span className={`text-xs px-2.5 py-1 rounded-full font-medium ${
                    files.length >= MAX_FILES 
                      ? 'bg-[#9E2B52] text-white border border-[#731A38]' 
                      : 'bg-[#F5E3E9] text-[#9E2B52] border border-[#E8C2CE]'
                  }`}>
                    {files.length} / {MAX_FILES} fotos
                  </span>
                </div>

                {/* Dropzone */}
                <div
                  onDragOver={handleDragOver}
                  onDragLeave={handleDragLeave}
                  onDrop={handleDrop}
                  className={`border-2 border-dashed rounded-2xl p-6 text-center cursor-pointer transition flex flex-col items-center justify-center min-h-[160px] ${
                    isDragging
                      ? 'border-[#D4507D] bg-[#F5E3E9]/80'
                      : 'border-[#E8C2CE] hover:border-[#D4507D] bg-[#FAF0F3]/60'
                  }`}
                >
                  <input
                    type="file"
                    multiple
                    accept="image/*"
                    onChange={handleFileChange}
                    className="hidden"
                    id="file-upload-input"
                  />
                  <label htmlFor="file-upload-input" className="cursor-pointer flex flex-col items-center">
                    <div className="p-3 bg-[#F5E3E9] text-[#D4507D] rounded-full mb-3 border border-[#E8C2CE] shadow-xs">
                      <Upload className="w-5 h-5" />
                    </div>
                    <p className="text-xs font-semibold text-[#731A38] mb-1">
                      Arrastra tus imágenes o <span className="text-[#D4507D] underline">examina carpetas</span>
                    </p>
                    <p className="text-[11px] text-[#9E2B52]/70">Soporta hasta 100 fotos de cuadernos o textos</p>
                  </label>
                </div>

                {/* Previews */}
                {files.length > 0 && (
                  <div className="mt-4">
                    <div className="flex items-center justify-between mb-2">
                      <span className="text-xs font-semibold text-[#9E2B52]">Fotos seleccionadas</span>
                      <button 
                        onClick={clearAllFiles}
                        className="text-xs text-[#9E2B52] hover:text-[#731A38] flex items-center gap-1 font-medium underline"
                      >
                        <Trash2 className="w-3 h-3" /> Limpiar
                      </button>
                    </div>

                    <div className="grid grid-cols-4 sm:grid-cols-5 gap-2.5 max-h-52 overflow-y-auto p-1 scrollbar-thin">
                      {files.map((fileObj) => (
                        <div 
                          key={fileObj.id} 
                          className="relative group aspect-square rounded-xl overflow-hidden bg-[#F5E3E9] border border-[#E8C2CE] shadow-xs"
                        >
                          <img 
                            src={fileObj.preview} 
                            alt={fileObj.name} 
                            className="w-full h-full object-cover transition group-hover:scale-105"
                          />
                          <button
                            onClick={() => removeFile(fileObj.id)}
                            className="absolute top-1 right-1 p-1 bg-[#731A38]/80 hover:bg-[#9E2B52] text-white rounded-md opacity-0 group-hover:opacity-100 transition"
                          >
                            <X className="w-3 h-3" />
                          </button>
                        </div>
                      ))}
                    </div>
                  </div>
                )}
              </div>

              {/* Options */}
              <div className="bg-white/80 border border-[#E8C2CE] rounded-3xl p-5 shadow-lg shadow-[#E8C2CE]/20 backdrop-blur-sm flex flex-col gap-4">
                <div className="flex items-center gap-2 border-b border-[#F5E3E9] pb-3">
                  <Settings className="w-4 h-4 text-[#D4507D]" />
                  <h3 className="font-semibold text-[#731A38] text-xs uppercase tracking-wider">Ajustes del Esquema</h3>
                </div>

                {/* Depth */}
                <div>
                  <label className="block text-xs font-semibold text-[#9E2B52] mb-2">
                    Profundidad de Contenido
                  </label>
                  <div className="grid grid-cols-3 gap-2">
                    {[
                      { id: 'ejecutivo', name: 'Sintético', desc: 'Resumen' },
                      { id: 'detallado', name: 'Detallado', desc: 'Equilibrado' },
                      { id: 'exhaustivo', name: 'Exhaustivo', desc: 'Súper Completo' }
                    ].map(opt => (
                      <button
                        key={opt.id}
                        type="button"
                        onClick={() => setDepthLevel(opt.id)}
                        className={`p-2.5 rounded-2xl border text-left transition flex flex-col ${
                          depthLevel === opt.id
                            ? 'bg-[#9E2B52] text-white border-[#731A38] shadow-md shadow-[#9E2B52]/20'
                            : 'bg-[#FAF0F3] border-[#E8C2CE] text-[#731A38] hover:bg-[#F5E3E9]'
                        }`}
                      >
                        <span className="text-xs font-bold">{opt.name}</span>
                        <span className={`text-[10px] mt-0.5 ${depthLevel === opt.id ? 'text-[#F5E3E9]' : 'text-[#9E2B52]'}`}>{opt.desc}</span>
                      </button>
                    ))}
                  </div>
                </div>

                {/* Style */}
                <div>
                  <label className="block text-xs font-semibold text-[#9E2B52] mb-2">
                    Vista Preferida
                  </label>
                  <div className="grid grid-cols-2 gap-2">
                    {[
                      { id: 'mindmap', name: 'Mapa Conceptual', icon: BrainCircuit },
                      { id: 'outline', name: 'Esquema Jerárquico', icon: Layers },
                      { id: 'flashcards', name: 'Fichas de Estudio', icon: BookOpen },
                      { id: 'quiz', name: 'Cuestionario', icon: HelpCircle },
                    ].map(opt => {
                      const Icon = opt.icon;
                      return (
                        <button
                          key={opt.id}
                          type="button"
                          onClick={() => setOutputStyle(opt.id)}
                          className={`p-2.5 rounded-2xl border flex items-center gap-2 text-left transition ${
                            outputStyle === opt.id
                              ? 'bg-[#9E2B52] text-white border-[#731A38] shadow-md shadow-[#9E2B52]/20'
                              : 'bg-[#FAF0F3] border-[#E8C2CE] text-[#731A38] hover:bg-[#F5E3E9]'
                          }`}
                        >
                          <Icon className={`w-4 h-4 ${outputStyle === opt.id ? 'text-white' : 'text-[#D4507D]'}`} />
                          <span className="text-xs font-bold">{opt.name}</span>
                        </button>
                      );
                    })}
                  </div>
                </div>

                {/* Custom Focus */}
                <div>
                  <label className="block text-xs font-semibold text-[#9E2B52] mb-1.5">
                    Instrucciones Especiales
                  </label>
                  <textarea
                    value={customFocus}
                    onChange={(e) => setCustomFocus(e.target.value)}
                    placeholder="Ej. Priorizar definiciones, fechas históricas, fórmulas o clasificaciones..."
                    className="w-full bg-[#FAF0F3]/80 border border-[#E8C2CE] rounded-2xl p-3 text-xs text-[#4A1829] placeholder-[#9E2B52]/50 focus:outline-none focus:border-[#D4507D] resize-none h-20"
                  />
                </div>

                {/* Action */}
                <button
                  onClick={generateOutlineWithGemini}
                  disabled={files.length === 0 || isProcessing}
                  className={`w-full py-3.5 px-4 rounded-2xl font-bold text-xs uppercase tracking-wider flex items-center justify-center gap-2 transition shadow-md ${
                    files.length === 0 || isProcessing
                      ? 'bg-[#E8C2CE] text-[#9E2B52]/50 cursor-not-allowed'
                      : 'bg-gradient-to-r from-[#9E2B52] via-[#D4507D] to-[#E66792] text-white hover:opacity-95 shadow-[#D4507D]/30 active:scale-[0.99]'
                  }`}
                >
                  <Sparkles className="w-4 h-4" />
                  {isProcessing ? 'Generando Esquema Rose...' : 'Generar Esquema Rosa Francés'}
                </button>
              </div>

            </div>

            {/* Right Column: Guidance / Loading */}
            <div className="w-full lg:w-7/12 flex flex-col justify-center">
              {isProcessing ? (
                /* Loading State */
                <div className="bg-white/90 border border-[#E8C2CE] rounded-3xl p-8 shadow-xl backdrop-blur-md flex flex-col items-center justify-center text-center max-w-lg mx-auto w-full min-h-[380px]">
                  <div className="relative mb-6">
                    <div className="w-20 h-20 rounded-3xl bg-[#F5E3E9] border border-[#E8C2CE] flex items-center justify-center">
                      <Flower2 className="w-10 h-10 text-[#D4507D] animate-spin" style={{ animationDuration: '10s' }} />
                    </div>
                  </div>

                  <h3 className="text-base font-bold text-[#731A38] mb-1">Sintetizando tus Apuntes</h3>
                  <p className="text-xs text-[#9E2B52] mb-6 max-w-xs">{processingStatus}</p>

                  <div className="w-full bg-[#F5E3E9] rounded-full h-2.5 border border-[#E8C2CE] overflow-hidden mb-3">
                    <div 
                      className="bg-gradient-to-r from-[#9E2B52] to-[#E66792] h-full transition-all duration-300 rounded-full"
                      style={{ width: `${processingProgress}%` }}
                    />
                  </div>
                  <span className="text-xs font-semibold text-[#9E2B52]">{processingProgress}%</span>
                </div>
              ) : errorMessage ? (
                /* Error */
                <div className="bg-white border border-[#E8C2CE] rounded-3xl p-6 text-center max-w-lg mx-auto shadow-md">
                  <AlertCircle className="w-10 h-10 text-[#9E2B52] mx-auto mb-3" />
                  <h3 className="text-sm font-bold text-[#731A38] mb-1">No se pudo completar la síntesis</h3>
                  <p className="text-xs text-[#9E2B52] mb-4">{errorMessage}</p>
                  <button
                    onClick={() => setErrorMessage(null)}
                    className="px-4 py-2 bg-[#9E2B52] hover:bg-[#731A38] text-white rounded-xl text-xs font-semibold"
                  >
                    Reintentar
                  </button>
                </div>
              ) : (
                /* Visual Welcome Guidance */
                <div className="bg-white/70 border border-[#E8C2CE] rounded-3xl p-8 sm:p-12 text-center flex flex-col items-center justify-center relative overflow-hidden shadow-xl shadow-[#E8C2CE]/10">
                  <div className="p-4 bg-[#F5E3E9] rounded-3xl border border-[#E8C2CE] text-[#D4507D] mb-6 shadow-xs">
                    <Flower2 className="w-8 h-8" />
                  </div>

                  <h2 className="text-2xl sm:text-3xl font-extrabold text-[#731A38] mb-3 tracking-tight">
                    Esquemas Académicos en Rosa Francés
                  </h2>
                  <p className="text-xs sm:text-sm text-[#9E2B52] max-w-md mb-8 leading-relaxed">
                    Sube hasta 100 fotos de tus cuadernos o textos. La IA analizará la estructura para generar mapas conceptuales interactivos, fichas y documentos guardables en PDF.
                  </p>

                  <div className="grid grid-cols-1 sm:grid-cols-3 gap-4 w-full max-w-lg text-left">
                    <div className="p-4 rounded-2xl bg-white border border-[#E8C2CE] shadow-xs">
                      <div className="text-xs font-bold text-[#731A38] mb-1">1. Fotos de Cuadernos</div>
                      <p className="text-[11px] text-[#9E2B52]/80">Procesa hasta 100 imágenes por tema.</p>
                    </div>
                    <div className="p-4 rounded-2xl bg-white border border-[#E8C2CE] shadow-xs">
                      <div className="text-xs font-bold text-[#731A38] mb-1">2. Red de Conceptos</div>
                      <p className="text-[11px] text-[#9E2B52]/80">Sintetiza árbol conceptual y notas.</p>
                    </div>
                    <div className="p-4 rounded-2xl bg-white border border-[#E8C2CE] shadow-xs">
                      <div className="text-xs font-bold text-[#731A38] mb-1">3. Guardar en PDF</div>
                      <p className="text-[11px] text-[#9E2B52]/80">Exporta documento listo para imprimir.</p>
                    </div>
                  </div>
                </div>
              )}
            </div>

          </div>
        ) : (
          /* RESULT DISPLAY VIEW */
          <div className="flex-1 flex flex-col overflow-hidden bg-[#FAF0F3]/40">
            
            {/* Toolbar */}
            <div className="no-print border-b border-[#E8C2CE] bg-white/90 px-4 lg:px-8 py-3 flex flex-wrap items-center justify-between gap-4 shadow-xs">
              
              <div>
                <h2 className="text-base font-extrabold text-[#731A38] flex items-center gap-2">
                  <span>{resultData.title}</span>
                  <span className="text-[10px] font-bold px-2.5 py-0.5 rounded-full bg-[#F5E3E9] text-[#9E2B52] border border-[#E8C2CE] uppercase">
                    {depthLevel}
                  </span>
                </h2>
                <p className="text-xs text-[#9E2B52]/80 line-clamp-1 max-w-xl">{resultData.summary}</p>
              </div>

              {/* View Switcher Tabs */}
              <div className="flex items-center gap-1 bg-[#F5E3E9] p-1 rounded-2xl border border-[#E8C2CE]">
                {[
                  { id: 'mindmap', label: 'Mapa Conceptual', icon: BrainCircuit },
                  { id: 'outline', label: 'Esquema Texto', icon: Layers },
                  { id: 'flashcards', label: `Fichas (${resultData.flashcards?.length || 0})`, icon: BookOpen },
                  { id: 'quiz', label: `Cuestionario (${resultData.quiz?.length || 0})`, icon: HelpCircle }
                ].map(tab => {
                  const Icon = tab.icon;
                  return (
                    <button
                      key={tab.id}
                      onClick={() => setActiveTab(tab.id)}
                      className={`flex items-center gap-2 px-3 py-1.5 rounded-xl text-xs font-bold transition ${
                        activeTab === tab.id
                          ? 'bg-[#9E2B52] text-white shadow-xs'
                          : 'text-[#731A38] hover:bg-white/60'
                      }`}
                    >
                      <Icon className="w-3.5 h-3.5" />
                      <span>{tab.label}</span>
                    </button>
                  );
                })}
              </div>

              {/* PDF & Export Actions */}
              <div className="flex items-center gap-2">
                <button
                  onClick={exportPDF}
                  className="flex items-center gap-1.5 px-3.5 py-1.5 text-xs font-bold bg-[#9E2B52] hover:bg-[#731A38] text-white rounded-xl shadow-xs transition"
                  title="Guardar en PDF o Imprimir"
                >
                  <Printer className="w-3.5 h-3.5" />
                  <span>Guardar PDF</span>
                </button>

                <button
                  onClick={copyAsText}
                  className="flex items-center gap-1.5 px-3 py-1.5 text-xs font-semibold bg-white hover:bg-[#F5E3E9] text-[#731A38] rounded-xl border border-[#E8C2CE] transition"
                >
                  {copied ? <Check className="w-3.5 h-3.5 text-emerald-600" /> : <Copy className="w-3.5 h-3.5 text-[#D4507D]" />}
                  <span>{copied ? 'Copiado' : 'Copiar'}</span>
                </button>

                <button
                  onClick={exportMarkdown}
                  className="flex items-center gap-1.5 px-3 py-1.5 text-xs font-semibold bg-white hover:bg-[#F5E3E9] text-[#731A38] rounded-xl border border-[#E8C2CE] transition"
                >
                  <Download className="w-3.5 h-3.5 text-[#D4507D]" />
                  <span>.MD</span>
                </button>
              </div>

            </div>

            {/* Content Printable PDF Target Area */}
            <div id="printable-pdf-area" className="flex-1 overflow-hidden relative">
              
              {/* TAB 1: VISUAL MIND MAP */}
              {activeTab === 'mindmap' && (
                <div className="w-full h-full flex flex-col lg:flex-row overflow-hidden relative">
                  
                  {/* Floating Controls */}
                  <div className="no-print absolute top-4 left-4 z-10 flex flex-col gap-2">
                    <div className="flex items-center gap-1 bg-white/90 border border-[#E8C2CE] rounded-2xl p-1 shadow-md backdrop-blur-md">
                      <button
                        onClick={() => setZoomLevel(prev => Math.min(prev + 0.15, 2))}
                        className="p-1.5 hover:bg-[#F5E3E9] text-[#731A38] rounded-xl transition"
                      >
                        <ZoomIn className="w-4 h-4" />
                      </button>
                      <button
                        onClick={() => setZoomLevel(1)}
                        className="px-2 py-1 text-xs text-[#731A38] hover:bg-[#F5E3E9] rounded-xl font-mono font-semibold"
                      >
                        {Math.round(zoomLevel * 100)}%
                      </button>
                      <button
                        onClick={() => setZoomLevel(prev => Math.max(prev - 0.15, 0.5))}
                        className="p-1.5 hover:bg-[#F5E3E9] text-[#731A38] rounded-xl transition"
                      >
                        <ZoomOut className="w-4 h-4" />
                      </button>
                    </div>

                    <div className="flex items-center bg-white/90 border border-[#E8C2CE] rounded-2xl px-3 py-1.5 shadow-md backdrop-blur-md w-60">
                      <Search className="w-3.5 h-3.5 text-[#D4507D] mr-2" />
                      <input
                        type="text"
                        placeholder="Buscar concepto..."
                        value={searchQuery}
                        onChange={(e) => setSearchQuery(e.target.value)}
                        className="bg-transparent text-xs text-[#4A1829] placeholder-[#9E2B52]/50 focus:outline-none w-full"
                      />
                    </div>
                  </div>

                  {/* Canvas Tree */}
                  <div className="flex-1 h-full overflow-auto p-12 flex items-center justify-center bg-[#FAF0F3]/20">
                    <div 
                      className="transition-transform duration-200 origin-center"
                      style={{ transform: `scale(${zoomLevel})` }}
                    >
                      <MindMapTree 
                        node={resultData.rootNode} 
                        selectedNode={selectedNode}
                        setSelectedNode={setSelectedNode}
                        searchQuery={searchQuery}
                        collapsedNodes={collapsedNodes}
                        toggleNodeCollapse={toggleNodeCollapse}
                      />
                    </div>
                  </div>

                  {/* Detail Panel */}
                  {selectedNode && (
                    <div className="no-print w-full lg:w-80 bg-white/95 border-t lg:border-t-0 lg:border-l border-[#E8C2CE] p-5 overflow-y-auto flex flex-col gap-4 shadow-xl z-20">
                      <div className="flex items-center justify-between border-b border-[#F5E3E9] pb-3">
                        <span className="text-[10px] uppercase font-bold tracking-wider text-[#9E2B52] bg-[#F5E3E9] px-2.5 py-0.5 rounded-full border border-[#E8C2CE]">
                          Detalle Conceptual
                        </span>
                        <button
                          onClick={() => setSelectedNode(null)}
                          className="p-1 text-[#9E2B52] hover:text-[#731A38] rounded-lg"
                        >
                          <X className="w-4 h-4" />
                        </button>
                      </div>

                      <div>
                        <h3 className="text-base font-bold text-[#731A38] mb-2">{selectedNode.label}</h3>
                        {selectedNode.description && (
                          <p className="text-xs text-[#4A1829] leading-relaxed bg-[#FAF0F3] p-3 rounded-2xl border border-[#E8C2CE] mb-3">
                            {selectedNode.description}
                          </p>
                        )}
                      </div>

                      {selectedNode.details && selectedNode.details.length > 0 && (
                        <div>
                          <h4 className="text-xs font-bold text-[#9E2B52] mb-2">Puntos de Examen</h4>
                          <ul className="space-y-2">
                            {selectedNode.details.map((detail, idx) => (
                              <li key={idx} className="flex items-start gap-2 text-xs text-[#4A1829] bg-[#FAF0F3]/60 p-2.5 rounded-xl border border-[#E8C2CE]/60">
                                <span className="w-1.5 h-1.5 rounded-full bg-[#D4507D] mt-1.5 shrink-0" />
                                <span>{detail}</span>
                              </li>
                            ))}
                          </ul>
                        </div>
                      )}
                    </div>
                  )}

                </div>
              )}

              {/* TAB 2: TEXT OUTLINE (PDF Export Optimized) */}
              {activeTab === 'outline' && (
                <div className="w-full h-full overflow-y-auto p-6 lg:p-12 max-w-4xl mx-auto">
                  <div className="bg-white border border-[#E8C2CE] rounded-3xl p-8 lg:p-10 shadow-lg">
                    <div className="border-b border-[#E8C2CE] pb-6 mb-6">
                      <h1 className="text-2xl font-extrabold text-[#731A38] mb-2">{resultData.title}</h1>
                      <p className="text-xs text-[#9E2B52] leading-relaxed">{resultData.summary}</p>
                    </div>

                    <OutlineNodeItem node={resultData.rootNode} depth={0} />
                  </div>
                </div>
              )}

              {/* TAB 3: FLASHCARDS */}
              {activeTab === 'flashcards' && (
                <div className="w-full h-full flex flex-col items-center justify-center p-6 max-w-2xl mx-auto">
                  {resultData.flashcards && resultData.flashcards.length > 0 ? (
                    <div className="w-full flex flex-col items-center">
                      
                      <div className="flex items-center justify-between w-full mb-4 px-2">
                        <span className="text-xs font-semibold text-[#9E2B52]">
                          Ficha {currentCardIndex + 1} de {resultData.flashcards.length}
                        </span>
                        {resultData.flashcards[currentCardIndex].category && (
                          <span className="text-xs px-2.5 py-0.5 rounded-full bg-[#F5E3E9] border border-[#E8C2CE] text-[#9E2B52] font-medium">
                            {resultData.flashcards[currentCardIndex].category}
                          </span>
                        )}
                      </div>

                      <div 
                        onClick={() => setIsCardFlipped(!isCardFlipped)}
                        className="w-full min-h-[280px] bg-white border-2 border-[#E8C2CE] hover:border-[#D4507D] rounded-3xl p-8 shadow-xl flex flex-col items-center justify-center text-center cursor-pointer transition-all duration-300 relative group"
                      >
                        <span className="no-print absolute top-4 right-4 text-[10px] text-[#9E2B52] font-semibold uppercase tracking-wider">
                          {isCardFlipped ? 'Respuesta' : 'Haz clic para voltear'}
                        </span>

                        <div className="flex-1 flex items-center justify-center">
                          {isCardFlipped ? (
                            <p className="text-base font-semibold text-[#731A38] leading-relaxed">
                              {resultData.flashcards[currentCardIndex].answer}
                            </p>
                          ) : (
                            <p className="text-lg font-bold text-[#4A1829] leading-relaxed">
                              {resultData.flashcards[currentCardIndex].question}
                            </p>
                          )}
                        </div>

                        <div className="no-print text-xs text-[#9E2B52] flex items-center gap-1 mt-4 font-medium">
                          <RefreshCw className="w-3.5 h-3.5" /> Voltear tarjeta
                        </div>
                      </div>

                      <div className="no-print flex items-center justify-between w-full mt-6">
                        <button
                          onClick={() => {
                            setIsCardFlipped(false);
                            setCurrentCardIndex(prev => Math.max(0, prev - 1));
                          }}
                          disabled={currentCardIndex === 0}
                          className="px-4 py-2 bg-white hover:bg-[#F5E3E9] disabled:opacity-40 text-[#731A38] text-xs font-semibold rounded-xl border border-[#E8C2CE] transition"
                        >
                          Anterior
                        </button>

                        <button
                          onClick={() => setIsCardFlipped(!isCardFlipped)}
                          className="px-4 py-2 bg-[#F5E3E9] text-[#9E2B52] hover:bg-[#E8C2CE] text-xs font-semibold rounded-xl border border-[#E8C2CE] transition"
                        >
                          {isCardFlipped ? 'Ver Pregunta' : 'Ver Respuesta'}
                        </button>

                        <button
                          onClick={() => {
                            setIsCardFlipped(false);
                            setCurrentCardIndex(prev => Math.min(resultData.flashcards.length - 1, prev + 1));
                          }}
                          disabled={currentCardIndex === resultData.flashcards.length - 1}
                          className="px-4 py-2 bg-white hover:bg-[#F5E3E9] disabled:opacity-40 text-[#731A38] text-xs font-semibold rounded-xl border border-[#E8C2CE] transition"
                        >
                          Siguiente
                        </button>
                      </div>

                    </div>
                  ) : (
                    <p className="text-[#9E2B52] text-xs">No hay fichas disponibles.</p>
                  )}
                </div>
              )}

              {/* TAB 4: QUIZ */}
              {activeTab === 'quiz' && (
                <div className="w-full h-full overflow-y-auto p-6 lg:p-12 max-w-3xl mx-auto">
                  {resultData.quiz && resultData.quiz.length > 0 ? (
                    <div className="space-y-6">
                      
                      {quizSubmitted && (
                        <div className="bg-white border-2 border-[#E8C2CE] rounded-3xl p-6 text-center shadow-md">
                          <Award className="w-10 h-10 text-[#D4507D] mx-auto mb-2" />
                          <h3 className="text-base font-bold text-[#731A38] mb-1">Evaluación Finalizada</h3>
                          <p className="text-xs text-[#9E2B52]">
                            Puntuación: <span className="font-extrabold text-[#731A38]">{quizScore}</span> de {resultData.quiz.length} ({Math.round((quizScore / resultData.quiz.length) * 100)}%)
                          </p>
                          <button
                            onClick={resetQuiz}
                            className="no-print mt-4 px-4 py-2 bg-[#9E2B52] hover:bg-[#731A38] text-white rounded-xl text-xs font-semibold transition"
                          >
                            Repetir Test
                          </button>
                        </div>
                      )}

                      {resultData.quiz.map((q, idx) => (
                        <div key={q.id} className="bg-white border border-[#E8C2CE] rounded-3xl p-6 shadow-sm">
                          <h3 className="text-xs font-bold text-[#731A38] mb-4 flex items-start gap-2">
                            <span className="text-[#D4507D]">{idx + 1}.</span>
                            <span>{q.question}</span>
                          </h3>

                          <div className="space-y-2 mb-4">
                            {q.options.map((opt, optIdx) => {
                              const isSelected = quizAnswers[q.id] === optIdx;
                              const isCorrect = q.correctIndex === optIdx;
                              
                              let optionStyle = "bg-[#FAF0F3]/40 border-[#E8C2CE] text-[#4A1829] hover:bg-[#F5E3E9]";
                              if (quizSubmitted) {
                                if (isCorrect) optionStyle = "bg-emerald-50 border-emerald-400 text-emerald-900 font-semibold";
                                else if (isSelected) optionStyle = "bg-rose-50 border-rose-400 text-rose-900 font-semibold";
                              } else if (isSelected) {
                                optionStyle = "bg-[#9E2B52] text-white font-semibold border-[#731A38] shadow-xs";
                              }

                              return (
                                <button
                                  key={optIdx}
                                  onClick={() => handleQuizOptionSelect(q.id, optIdx)}
                                  disabled={quizSubmitted}
                                  className={`w-full text-left p-3 rounded-2xl border text-xs transition flex items-center justify-between ${optionStyle}`}
                                >
                                  <span>{opt}</span>
                                  {quizSubmitted && isCorrect && <CheckCircle2 className="w-4 h-4 text-emerald-600 shrink-0 ml-2" />}
                                  {quizSubmitted && isSelected && !isCorrect && <XCircle className="w-4 h-4 text-rose-600 shrink-0 ml-2" />}
                                </button>
                              );
                            })}
                          </div>

                          {quizSubmitted && q.explanation && (
                            <div className="bg-[#FAF0F3] p-3 rounded-2xl border border-[#E8C2CE] text-xs text-[#731A38] leading-relaxed">
                              <strong className="text-[#9E2B52]">Explicación:</strong> {q.explanation}
                            </div>
                          )}
                        </div>
                      ))}

                      {!quizSubmitted && (
                        <button
                          onClick={submitQuiz}
                          disabled={Object.keys(quizAnswers).length === 0}
                          className="no-print w-full py-3 bg-[#9E2B52] hover:bg-[#731A38] disabled:opacity-40 text-white rounded-2xl text-xs font-bold uppercase tracking-wider transition shadow-md"
                        >
                          Calcular Puntuación
                        </button>
                      )}

                    </div>
                  ) : (
                    <p className="text-[#9E2B52] text-xs">No hay cuestionario generado.</p>
                  )}
                </div>
              )}

            </div>
          </div>
        )}
      </main>
    </div>
  );
}

// Subcomponent: French Rose Tree Nodes
function MindMapTree({ node, selectedNode, setSelectedNode, searchQuery, collapsedNodes, toggleNodeCollapse }) {
  if (!node) return null;

  const isSelected = selectedNode?.id === node.id;
  const isCollapsed = collapsedNodes[node.id];
  const isMatch = searchQuery && node.label.toLowerCase().includes(searchQuery.toLowerCase());

  return (
    <div className="flex flex-col items-center">
      <div 
        onClick={() => setSelectedNode(node)}
        className={`p-3.5 rounded-2xl border transition-all duration-200 cursor-pointer min-w-[180px] max-w-[250px] text-center shadow-md relative group ${
          isSelected 
            ? 'bg-[#9E2B52] text-white border-[#731A38] ring-4 ring-[#E8C2CE] scale-105' 
            : isMatch
            ? 'bg-[#F5E3E9] border-[#D4507D] text-[#731A38] ring-2 ring-[#E8C2CE]'
            : 'bg-white border-[#E8C2CE] text-[#4A1829] hover:border-[#D4507D] hover:bg-[#FAF0F3]'
        }`}
      >
        <h4 className="text-xs font-bold leading-snug">{node.label}</h4>
        {node.description && (
          <p className="text-[10px] opacity-80 line-clamp-2 mt-1 font-normal">
            {node.description}
          </p>
        )}

        {node.children && node.children.length > 0 && (
          <button
            onClick={(e) => {
              e.stopPropagation();
              toggleNodeCollapse(node.id);
            }}
            className="no-print absolute -bottom-2.5 left-1/2 -translate-x-1/2 w-5 h-5 rounded-full bg-white border border-[#E8C2CE] text-[#9E2B52] flex items-center justify-center hover:bg-[#9E2B52] hover:text-white text-[10px] font-bold transition shadow-xs"
          >
            {isCollapsed ? '+' : '-'}
          </button>
        )}
      </div>

      {node.children && node.children.length > 0 && !isCollapsed && (
        <div className="flex flex-col items-center mt-6">
          <div className="w-0.5 h-6 bg-[#E8C2CE]" />
          <div className="flex items-start gap-6 relative">
            {node.children.map((childNode, index) => (
              <div key={childNode.id || index} className="flex flex-col items-center relative">
                <div className="w-0.5 h-6 bg-[#E8C2CE]" />
                <MindMapTree 
                  node={childNode} 
                  selectedNode={selectedNode}
                  setSelectedNode={setSelectedNode}
                  searchQuery={searchQuery}
                  collapsedNodes={collapsedNodes}
                  toggleNodeCollapse={toggleNodeCollapse}
                />
              </div>
            ))}
          </div>
        </div>
      )}
    </div>
  );
}

// Subcomponent: Outline French Rose Items
function OutlineNodeItem({ node, depth = 0 }) {
  const [isOpen, setIsOpen] = useState(true);
  const hasChildren = node.children && node.children.length > 0;

  return (
    <div className="mb-4">
      <div 
        className={`flex items-start gap-2 ${depth === 0 ? 'text-base font-extrabold text-[#731A38]' : depth === 1 ? 'text-sm font-bold text-[#9E2B52]' : 'text-xs font-semibold text-[#4A1829]'}`}
        style={{ paddingLeft: `${depth * 1.25}rem` }}
      >
        {hasChildren ? (
          <button 
            onClick={() => setIsOpen(!isOpen)}
            className="p-1 rounded hover:bg-[#F5E3E9] text-[#D4507D] mt-0.5 no-print"
          >
            {isOpen ? <ChevronDown className="w-4 h-4" /> : <ChevronRight className="w-4 h-4" />}
          </button>
        ) : (
          <span className="w-2 h-2 rounded-full bg-[#D4507D] mt-2 mx-1.5 shrink-0" />
        )}

        <div className="flex-1">
          <div>{node.label}</div>
          {node.description && (
            <p className="text-xs text-[#9E2B52] font-normal mt-0.5 leading-relaxed">
              {node.description}
            </p>
          )}

          {node.details && node.details.length > 0 && (
            <ul className="mt-2 space-y-1">
              {node.details.map((detail, idx) => (
                <li key={idx} className="text-xs text-[#4A1829] bg-[#FAF0F3] px-3 py-1.5 rounded-xl border border-[#E8C2CE] font-normal">
                  {detail}
                </li>
              ))}
            </ul>
          )}
        </div>
      </div>

      {hasChildren && isOpen && (
        <div className="mt-3 space-y-2 border-l-2 border-[#E8C2CE] ml-3">
          {node.children.map((child, idx) => (
            <OutlineNodeItem key={child.id || idx} node={child} depth={depth + 1} />
          ))}
        </div>
      )}
    </div>
  );
}
