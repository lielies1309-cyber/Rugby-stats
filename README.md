
import React, { useState, useRef, useEffect } from 'react';
import { Camera, Upload, Play, Info, CheckCircle, AlertCircle, Loader2, ChevronRight } from 'lucide-react';

const apiKey = ""; // Environment provides this at runtime

const App = () => {
  const [videoFile, setVideoFile] = useState(null);
  const [videoPreview, setVideoPreview] = useState(null);
  const [isAnalyzing, setIsAnalyzing] = useState(false);
  const [analysis, setAnalysis] = useState(null);
  const [error, setError] = useState(null);
  const videoRef = useRef(null);

  const handleFileUpload = (e) => {
    const file = e.target.files[0];
    if (file && file.type.startsWith('video/')) {
      setVideoFile(file);
      setVideoPreview(URL.createObjectURL(file));
      setAnalysis(null);
      setError(null);
    } else {
      setError("Please upload a valid video file.");
    }
  };

  const fileToBase64 = (file) => {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.readAsDataURL(file);
      reader.onload = () => resolve(reader.result.split(',')[1]);
      reader.onerror = (error) => reject(error);
    });
  };

  const analyzeKick = async () => {
    if (!videoFile) return;

    setIsAnalyzing(true);
    setError(null);

    try {
      // Capture a frame for analysis since Gemini Flash understands images well
      // In a real mobile environment, we'd send segments, but here we'll analyze the setup/swing frame
      const video = videoRef.current;
      const canvas = document.createElement('canvas');
      canvas.width = video.videoWidth;
      canvas.height = video.videoHeight;
      const ctx = canvas.getContext('2d');
      
      // Draw the current frame or middle of the video
      video.currentTime = video.duration / 2; 
      await new Promise(r => setTimeout(r, 500)); // Wait for seek
      ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
      const base64Image = canvas.toDataURL('image/jpeg').split(',')[1];

      const prompt = `Analyze this rugby kick technique. Focus on: 1. Body alignment 2. Plant foot positioning 3. Leg swing arc 4. Follow through. Provide a structured response with "Strengths", "Areas for Improvement", and a "Pro Tip". Keep it encouraging and technical.`;

      let attempt = 0;
      let result = null;
      
      while (attempt < 5) {
        try {
          const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
              contents: [{
                parts: [
                  { text: prompt },
                  { inlineData: { mimeType: "image/jpeg", data: base64Image } }
                ]
              }]
            })
          });

          if (!response.ok) throw new Error('API request failed');
          result = await response.json();
          break;
        } catch (e) {
          attempt++;
          if (attempt === 5) throw e;
          await new Promise(r => setTimeout(r, Math.pow(2, attempt) * 1000));
        }
      }

      const textAnalysis = result.candidates?.[0]?.content?.parts?.[0]?.text;
      if (textAnalysis) {
        setAnalysis(textAnalysis);
      } else {
        throw new Error("Could not parse analysis.");
      }
    } catch (err) {
      setError("Analysis failed. Please try a clearer video or check your connection.");
      console.error(err);
    } finally {
      setIsAnalyzing(false);
    }
  };

  return (
    <div className="min-h-screen bg-slate-50 text-slate-900 font-sans p-4 md:p-8">
      <div className="max-w-3xl mx-auto">
        {/* Header */}
        <header className="mb-8 text-center">
          <div className="inline-flex items-center justify-center p-3 bg-blue-600 rounded-2xl mb-4 text-white shadow-lg">
            <Camera size={32} />
          </div>
          <h1 className="text-3xl font-bold tracking-tight">Rugby Kick Coach</h1>
          <p className="text-slate-500 mt-2 text-lg">Upload your kicking video for instant technical feedback</p>
        </header>

        {/* Main Interface */}
        <div className="bg-white rounded-3xl shadow-xl overflow-hidden border border-slate-100">
          {!videoPreview ? (
            <div className="p-12 flex flex-col items-center justify-center border-2 border-dashed border-slate-200 m-4 rounded-2xl bg-slate-50">
              <Upload className="text-blue-500 mb-4" size={48} />
              <p className="text-lg font-medium text-slate-700">Drop your video here</p>
              <p className="text-sm text-slate-400 mb-6 text-center max-w-xs">
                Make sure the camera is at waist-height and captures your full movement.
              </p>
              <label className="cursor-pointer bg-blue-600 hover:bg-blue-700 text-white px-8 py-3 rounded-xl font-semibold transition-all shadow-md active:scale-95">
                Select Video
                <input type="file" className="hidden" accept="video/*" onChange={handleFileUpload} />
              </label>
            </div>
          ) : (
            <div className="p-4 md:p-6">
              <div className="relative rounded-2xl overflow-hidden bg-black shadow-inner aspect-video mb-6">
                <video 
                  ref={videoRef}
                  src={videoPreview} 
                  className="w-full h-full object-contain"
                  controls
                />
              </div>

              <div className="flex flex-col sm:flex-row gap-4 mb-8">
                <button 
                  onClick={analyzeKick}
                  disabled={isAnalyzing}
                  className={`flex-1 flex items-center justify-center gap-2 py-4 rounded-2xl font-bold text-lg transition-all shadow-lg ${
                    isAnalyzing ? 'bg-slate-100 text-slate-400' : 'bg-blue-600 text-white hover:bg-blue-700 active:scale-95'
                  }`}
                >
                  {isAnalyzing ? (
                    <>
                      <Loader2 className="animate-spin" />
                      Analyzing Form...
                    </>
                  ) : (
                    <>
                      <CheckCircle size={20} />
                      Analyze My Technique
                    </>
                  )}
                </button>
                <button 
                  onClick={() => {setVideoPreview(null); setAnalysis(null);}}
                  className="px-6 py-4 rounded-2xl font-semibold text-slate-600 bg-slate-100 hover:bg-slate-200 transition-colors"
                >
                  Clear
                </button>
              </div>

              {error && (
                <div className="mb-6 p-4 bg-red-50 text-red-700 rounded-xl flex items-start gap-3 border border-red-100">
                  <AlertCircle className="shrink-0 mt-0.5" />
                  <p>{error}</p>
                </div>
              )}

              {analysis && (
                <div className="bg-slate-50 rounded-2xl p-6 border border-slate-200 animate-in fade-in slide-in-from-bottom-4 duration-500">
                  <div className="flex items-center gap-2 mb-4 text-blue-700">
                    <Info size={24} />
                    <h2 className="text-xl font-bold">Coach Analysis</h2>
                  </div>
                  <div className="prose prose-slate max-w-none">
                    {analysis.split('\n').map((line, i) => (
                      <p key={i} className="mb-2 text-slate-700 leading-relaxed">
                        {line}
                      </p>
                    ))}
                  </div>
                </div>
              )}
            </div>
          )}
        </div>

        {/* Tips Section */}
        <section className="mt-12 grid grid-cols-1 md:grid-cols-2 gap-6">
          <div className="p-6 bg-white rounded-2xl border border-slate-100 shadow-sm">
            <h3 className="font-bold text-lg mb-3 flex items-center gap-2">
              <ChevronRight className="text-blue-500" size={18} />
              Best Angles
            </h3>
            <p className="text-slate-600 text-sm">
              For the best results, film from a side-on perspective (profile) or directly behind the kicker (to see ball flight).
            </p>
          </div>
          <div className="p-6 bg-white rounded-2xl border border-slate-100 shadow-sm">
            <h3 className="font-bold text-lg mb-3 flex items-center gap-2">
              <ChevronRight className="text-blue-500" size={18} />
              Stability
            </h3>
            <p className="text-slate-600 text-sm">
              Keep the camera steady. Use a tripod or lean it against your water bottle if you're training solo.
            </p>
          </div>
        </section>
      </div>
    </div>
  );
};

export default App;
