# CodeAlpha_Project-Task-3-Music-Generation-with-AI
Author- Sakshi Shukla


import React, { useState, useEffect, useRef } from "react";
import { 
  Play, 
  Pause, 
  Square, 
  RotateCcw, 
  Download, 
  Cpu, 
  FileCode, 
  Sliders, 
  Volume2, 
  Terminal, 
  Music, 
  Activity, 
  Sparkles, 
  Copy, 
  Check, 
  HelpCircle, 
  Info, 
  Eye, 
  TrendingDown, 
  FileAudio
} from "lucide-react";
import { MidiNote, SongPreset } from "./types";
import { getSongPresets, PythonCodeSnippets, noteToFreqs, generateCustomSong } from "./data";

export default function App() {
  // Tabs & Preferences
  const [activeTab, setActiveTab] = useState<"composer" | "training" | "code">("composer");
  const [selectedSongId, setSelectedSongId] = useState<string>("classical-chopin");
  const [instrumentType, setInstrumentType] = useState<"piano" | "synth" | "organ">("piano");
  const [volume, setVolume] = useState<number>(0.6);
  const [tempo, setTempo] = useState<number>(90);
  
  // Custom Generation Settings
  const [temperature, setTemperature] = useState<number>(0.85);
  const [seqLength, setSeqLength] = useState<number>(100);
  const [genLength, setGenLength] = useState<number>(150);
  const [generationStyle, setGenerationStyle] = useState<string>("classical");
  const [isGenerating, setIsGenerating] = useState<boolean>(false);
  const [generationStep, setGenerationStep] = useState<string>("");

  // Playback Engine State
  const [isPlaying, setIsPlaying] = useState<boolean>(false);
  const [playbackTime, setPlaybackTime] = useState<number>(0); // Current beat offset
  const [activeNotes, setActiveNotes] = useState<Set<number>>(new Set());

  // Training Simulator State
  const [isTraining, setIsTraining] = useState<boolean>(false);
  const [simEpoch, setSimEpoch] = useState<number>(0);
  const [simLoss, setSimLoss] = useState<number>(4.82);
  const [simAccuracy, setSimAccuracy] = useState<number>(0.02);
  const [simValLoss, setSimValLoss] = useState<number>(4.91);
  const [simValAccuracy, setSimValAccuracy] = useState<number>(0.02);
  const [simHistory, setSimHistory] = useState<any[]>([]);
  const [simLogs, setSimLogs] = useState<string[]>([]);
  const [simSpeed, setSimSpeed] = useState<"normal" | "fast">("normal");

  // UI state
  const [copiedIndex, setCopiedIndex] = useState<number | null>(null);
  const [showMLModal, setShowMLModal] = useState<boolean>(false);
  
  // Static Songs list
  const [songsList, setSongsList] = useState<SongPreset[]>([]);

  // Web Audio Refs
  const audioCtxRef = useRef<AudioContext | null>(null);
  const analyserRef = useRef<AnalyserNode | null>(null);
  const activeOscillatorsRef = useRef<any[]>([]);
  
  // Canvas Refs
  const pianoRollCanvasRef = useRef<HTMLCanvasElement | null>(null);
  const waveformCanvasRef = useRef<HTMLCanvasElement | null>(null);

  // Playback Loop Refs
  const animationFrameIdRef = useRef<number | null>(null);
  const startTimeRef = useRef<number>(0); // Real world start timestamp (ms)
  const startBeatRef = useRef<number>(0); // Beat offset when started
  const playedNotesSetRef = useRef<Set<string>>(new Set()); // unique key: "noteIndex-time"
  const currentSongRef = useRef<SongPreset | null>(null);
  const isPlayingRef = useRef<boolean>(false);

  // Initialize presets
  useEffect(() => {
    const presets = getSongPresets();
    setSongsList(presets);
  }, []);

  const currentSong = songsList.find(s => s.id === selectedSongId) || songsList[0];

  // Sync ref to prevent state-stretching inside standard animation loop
  useEffect(() => {
    currentSongRef.current = currentSong;
  }, [currentSong]);

  useEffect(() => {
    isPlayingRef.current = isPlaying;
  }, [isPlaying]);

  // Adjust tempo when changing tracks
  useEffect(() => {
    if (currentSong) {
      if (currentSong.id === "cyber-synth") {
        setTempo(125);
      } else if (currentSong.id === "early-chaos") {
        setTempo(80);
      } else if (currentSong.id === "developing") {
        setTempo(95);
      } else {
        setTempo(85);
      }
    }
  }, [selectedSongId, songsList]);

  // Init Web Audio
  const initAudio = () => {
    if (!audioCtxRef.current) {
      // Support standard web browsers
      const AudioCtxClass = window.AudioContext || (window as any).webkitAudioContext;
      audioCtxRef.current = new AudioCtxClass();
      
      analyserRef.current = audioCtxRef.current.createAnalyser();
      analyserRef.current.fftSize = 256;
      analyserRef.current.connect(audioCtxRef.current.destination);
    }
    if (audioCtxRef.current.state === 'suspended') {
      audioCtxRef.current.resume();
    }
  };

  // Trigger oscillator note
  const playSynthesizerNotes = (freqs: number[], durationSec: number) => {
    if (!audioCtxRef.current || !analyserRef.current) return;
    const ctx = audioCtxRef.current;
    const now = ctx.currentTime;

    freqs.forEach(freq => {
      const osc = ctx.createOscillator();
      const gainNode = ctx.createGain();

      osc.connect(gainNode);
      gainNode.connect(analyserRef.current!);

      // Map wave parameters based on instrument selection
      if (instrumentType === "piano") {
        osc.type = "sine";
        // Warm exponential pluck envelope
        const vol = (volume * 0.20) / freqs.length;
        gainNode.gain.setValueAtTime(0, now);
        gainNode.gain.linearRampToValueAtTime(vol, now + 0.02);
        gainNode.gain.exponentialRampToValueAtTime(0.001, now + durationSec);
      } else if (instrumentType === "synth") {
        osc.type = "sawtooth";
        // Quick biting envelope
        const vol = (volume * 0.12) / freqs.length;
        gainNode.gain.setValueAtTime(0, now);
        gainNode.gain.linearRampToValueAtTime(vol, now + 0.01);
        gainNode.gain.exponentialRampToValueAtTime(0.001, now + durationSec);
      } else {
        osc.type = "triangle";
        // Sustained organ envelope
        const vol = (volume * 0.15) / freqs.length;
        gainNode.gain.setValueAtTime(0, now);
        gainNode.gain.linearRampToValueAtTime(vol, now + 0.05);
        gainNode.gain.linearRampToValueAtTime(vol * 0.5, now + durationSec * 0.6);
        gainNode.gain.exponentialRampToValueAtTime(0.001, now + durationSec);
      }

      osc.frequency.setValueAtTime(freq, now);
      osc.start(now);
      osc.stop(now + durationSec);

      activeOscillatorsRef.current.push({ osc, gainNode, stopTime: now + durationSec });
    });
  };

  // Clean up active oscillators
  const stopAllOscillators = () => {
    activeOscillatorsRef.current.forEach(item => {
      try {
        item.osc.stop();
      } catch (e) {}
    });
    activeOscillatorsRef.current = [];
  };

  // Playback clock frame
  const handlePlaybackFrame = (timestamp: number) => {
    if (!isPlayingRef.current || !currentSongRef.current) return;

    if (!startTimeRef.current) {
      startTimeRef.current = timestamp;
    }

    const elapsedSec = (timestamp - startTimeRef.current) / 1000;
    const beatsElapsed = elapsedSec * (tempo / 60);
    const currentBeat = startBeatRef.current + beatsElapsed;

    setPlaybackTime(currentBeat);

    // Filter notes that need to be played in this exact frame
    const activeMidiPitches = new Set<number>();
    
    currentSongRef.current.notes.forEach((note, index) => {
      const notePlayKey = `${index}-${note.time}`;
      
      // Determine if note is currently active (lit keys)
      if (currentBeat >= note.time && currentBeat < note.time + note.duration) {
        // Extract midi numbers
        parseNoteToMidi(note.note).forEach(midi => activeMidiPitches.add(midi));
      }

      // Check if note offset matches current beat to trigger synth plucks
      if (currentBeat >= note.time && !playedNotesSetRef.current.has(notePlayKey)) {
        // Double check we aren't triggering old notes if we seeked
        if (currentBeat - note.time < 0.5) {
          playedNotesSetRef.current.add(notePlayKey);
          
          // Calculate physical duration in real seconds based on current BPM
          const durationSec = note.duration * (60 / tempo);
          playSynthesizerNotes(note.frequency, durationSec);
        }
      }
    });

    setActiveNotes(activeMidiPitches);

    // Check if song finished (reached last note time)
    const lastNote = currentSongRef.current.notes[currentSongRef.current.notes.length - 1];
    const maxDuration = lastNote ? lastNote.time + lastNote.duration : 100;
    
    if (currentBeat >= maxDuration) {
      handleStop();
    } else {
      animationFrameIdRef.current = requestAnimationFrame(handlePlaybackFrame);
    }
  };

  // Toggle playback
  const handlePlayPause = () => {
    initAudio();
    if (isPlaying) {
      // Pause
      setIsPlaying(false);
      if (animationFrameIdRef.current) {
        cancelAnimationFrame(animationFrameIdRef.current);
      }
      stopAllOscillators();
    } else {
      // Play
      setIsPlaying(true);
      startTimeRef.current = 0;
      startBeatRef.current = playbackTime;
      animationFrameIdRef.current = requestAnimationFrame(handlePlaybackFrame);
    }
  };

  // Stop playback completely
  const handleStop = () => {
    setIsPlaying(false);
    if (animationFrameIdRef.current) {
      cancelAnimationFrame(animationFrameIdRef.current);
    }
    stopAllOscillators();
    setPlaybackTime(0);
    playedNotesSetRef.current.clear();
    setActiveNotes(new Set());
    startTimeRef.current = 0;
  };

  // Reset/Restart playback
  const handleRestart = () => {
    handleStop();
    setTimeout(() => {
      handlePlayPause();
    }, 100);
  };

  // Helper to parse note tokens to midi notes (C3=48, C4=60, C5=72, etc.)
  const parseNoteToMidi = (noteStr: string): number[] => {
    if (noteStr.includes('.')) {
      return noteStr.split('.').map(parseSingleNoteToMidiNum);
    }
    return [parseSingleNoteToMidiNum(noteStr)];
  };

  const parseSingleNoteToMidiNum = (n: string): number => {
    if (/^\d+$/.test(n)) {
      return 60 + parseInt(n, 10); // relative to C4
    }
    const match = n.match(/^([A-G]#?|B?|D?|E?|F?|G?|A?b?)(-?\d+)$/);
    if (!match) return 60;
    const pitchName = match[1];
    const octave = parseInt(match[2], 10);
    const pitchIndexes: {[key: string]: number} = {
      'C': 0, 'C#': 1, 'Db': 1, 'D': 2, 'D#': 3, 'Eb': 3, 'E': 4,
      'F': 5, 'F#': 6, 'Gb': 6, 'G': 7, 'G#': 8, 'Ab': 8, 'A': 9,
      'A#': 10, 'Bb': 10, 'B': 11
    };
    const semitone = pitchIndexes[pitchName] || 0;
    return 12 * (octave + 1) + semitone;
  };

  // Draw scrolling piano roll
  useEffect(() => {
    const canvas = pianoRollCanvasRef.current;
    if (!canvas || !currentSong) return;

    const ctx = canvas.getContext("2d");
    if (!ctx) return;

    // Responsive sizing
    const width = canvas.width;
    const height = canvas.height;

    // Clear canvas
    ctx.fillStyle = "#111827"; // Dark background slate-900
    ctx.fillRect(0, 0, width, height);

    const keyboardWidth = 60;
    const scrollableWidth = width - keyboardWidth;
    const pixelsPerBeat = 45; // scale factor
    
    // Scale pitches from MIDI 48 (C3) to 84 (C6) -> 36 semitones
    const minMidi = 48;
    const maxMidi = 84;
    const pitchRange = maxMidi - minMidi + 1;
    const rowHeight = height / pitchRange;

    // 1. Draw horizontal piano grid lines
    ctx.strokeStyle = "#1f2937"; // gray-800
    ctx.lineWidth = 1;
    for (let p = 0; p <= pitchRange; p++) {
      const y = height - (p * rowHeight);
      ctx.beginPath();
      ctx.moveTo(keyboardWidth, y);
      ctx.lineTo(width, y);
      ctx.stroke();

      // Color backgrounds for black keys (sharps/flats)
      const currentMidi = minMidi + p;
      const semitone = currentMidi % 12;
      const isBlackKey = [1, 3, 6, 8, 10].includes(semitone);
      if (isBlackKey && p < pitchRange) {
        ctx.fillStyle = "rgba(17, 24, 39, 0.4)";
        ctx.fillRect(keyboardWidth, height - ((p + 1) * rowHeight), scrollableWidth, rowHeight);
      }
    }

    // 2. Draw vertical beat markers relative to current playbackTime
    ctx.strokeStyle = "rgba(31, 41, 55, 0.5)"; // gray-800
    const startBeatVisible = Math.max(0, Math.floor(playbackTime - 2));
    const endBeatVisible = Math.ceil(playbackTime + (scrollableWidth / pixelsPerBeat));

    for (let b = startBeatVisible; b <= endBeatVisible; b++) {
      const x = keyboardWidth + (b - playbackTime) * pixelsPerBeat;
      if (x >= keyboardWidth) {
        ctx.beginPath();
        ctx.moveTo(x, 0);
        ctx.lineTo(x, height);
        ctx.stroke();

        // Draw small beat numbers on top row
        ctx.fillStyle = "#4b5563"; // gray-600
        ctx.font = "9px monospace";
        ctx.fillText(`b.${b}`, x + 4, 12);
      }
    }

    // 3. Draw note bars
    currentSong.notes.forEach(note => {
      const midiNotes = parseNoteToMidi(note.note);
      const isNoteActive = playbackTime >= note.time && playbackTime < note.time + note.duration;

      midiNotes.forEach(midiNum => {
        // Only draw if within our visual octave boundaries
        if (midiNum >= minMidi && midiNum <= maxMidi) {
          const pitchIndex = midiNum - minMidi;
          const y = height - ((pitchIndex + 1) * rowHeight);
          
          const x = keyboardWidth + (note.time - playbackTime) * pixelsPerBeat;
          const noteWidth = note.duration * pixelsPerBeat;

          if (x + noteWidth >= keyboardWidth && x <= width) {
            // Shadow / Rounded card boundaries
            ctx.fillStyle = isNoteActive 
              ? "#06b6d4" // Active glowing cyan-500
              : "#3b82f6"; // Inactive sky blue-500
            
            // Draw neat rounded rect-like block
            ctx.fillRect(Math.max(keyboardWidth, x), y + 1, noteWidth - (x < keyboardWidth ? (keyboardWidth - x) : 0) - 1, rowHeight - 2);

            // Add inner detail reflection
            ctx.fillStyle = "rgba(255, 255, 255, 0.15)";
            ctx.fillRect(Math.max(keyboardWidth, x) + 2, y + 2, 4, rowHeight - 4);
          }
        }
      });
    });

    // 4. Playhead alignment marker
    ctx.strokeStyle = "#ec4899"; // pink-500 magenta line
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(keyboardWidth, 0);
    ctx.lineTo(keyboardWidth, height);
    ctx.stroke();

    // 5. Beautiful piano keyboard outline on the left edge
    ctx.fillStyle = "#1f2937"; // base drawer gray-800
    ctx.fillRect(0, 0, keyboardWidth, height);

    for (let p = 0; p < pitchRange; p++) {
      const midiNum = minMidi + p;
      const semitone = midiNum % 12;
      const isBlackKey = [1, 3, 6, 8, 10].includes(semitone);
      const y = height - ((p + 1) * rowHeight);

      const isActive = activeNotes.has(midiNum);

      if (isBlackKey) {
        ctx.fillStyle = isActive ? "#06b6d4" : "#111827"; // active vs black key
        ctx.fillRect(0, y + 1, keyboardWidth - 18, rowHeight - 2);
      } else {
        ctx.fillStyle = isActive ? "#22d3ee" : "#f9fafb"; // active vs white key
        ctx.fillRect(0, y + 1, keyboardWidth - 2, rowHeight - 2);
        
        // draw key separator line
        ctx.fillStyle = "#d1d5db"; // gray-300
        ctx.fillRect(0, y + rowHeight - 1, keyboardWidth - 2, 1);
      }
    }
  }, [playbackTime, selectedSongId, songsList, activeNotes]);

  // Audio Oscilloscope Waveform Drawer
  useEffect(() => {
    const canvas = waveformCanvasRef.current;
    if (!canvas) return;

    const ctx = canvas.getContext("2d");
    if (!ctx) return;

    const width = canvas.width;
    const height = canvas.height;

    let localFrameId: number;
    const dataArray = new Uint8Array(128);

    const drawWave = () => {
      ctx.clearRect(0, 0, width, height);

      // Gradient background line
      ctx.fillStyle = "rgba(17, 24, 39, 0.25)";
      ctx.fillRect(0, 0, width, height);

      if (analyserRef.current && isPlaying) {
        analyserRef.current.getByteTimeDomainData(dataArray);

        ctx.lineWidth = 2;
        ctx.strokeStyle = "#06b6d4"; // glowing neon cyan
        ctx.beginPath();

        const sliceWidth = width / dataArray.length;
        let x = 0;

        for (let i = 0; i < dataArray.length; i++) {
          const v = dataArray[i] / 128.0;
          const y = (v * height) / 2;

          if (i === 0) {
            ctx.moveTo(x, y);
          } else {
            ctx.lineTo(x, y);
          }

          x += sliceWidth;
        }

        ctx.lineTo(width, height / 2);
        ctx.stroke();
      } else {
        // Draw flat line
        ctx.strokeStyle = "#4b5563"; // gray-600
        ctx.lineWidth = 1.5;
        ctx.beginPath();
        ctx.moveTo(0, height / 2);
        ctx.lineTo(width, height / 2);
        ctx.stroke();
      }

      localFrameId = requestAnimationFrame(drawWave);
    };

    drawWave();

    return () => {
      cancelAnimationFrame(localFrameId);
    };
  }, [isPlaying]);

  // Copy Code snippet to clipboard helper
  const handleCopyCode = (codeText: string, index: number) => {
    navigator.clipboard.writeText(codeText);
    setCopiedIndex(index);
    setTimeout(() => {
      setCopiedIndex(null);
    }, 2000);
  };

  // Run Training Simulator loops
  const startSimulatedTraining = () => {
    if (isTraining) {
      setIsTraining(false);
      return;
    }

    setIsTraining(true);
    setSimEpoch(1);
    setSimLoss(4.82);
    setSimAccuracy(0.02);
    setSimValLoss(4.91);
    setSimValAccuracy(0.02);
    setSimHistory([]);
    setSimLogs([
      "[SYSTEM] Loading training data sequence vectors...",
      `[SYSTEM] Parsed 24 classical MIDI tracks inside 'midi_songs/' directory.`,
      `[SYSTEM] Extracted 42,912 individual notes and chord strings.`,
      `[SYSTEM] Vocabulary size: 142 unique tokens mapping pitches and triads.`,
      `[SYSTEM] Preparing training tensor inputs with SEQUENCE_LENGTH = ${seqLength}...`,
      "[SYSTEM] Output maps built! Compiled 42,812 patterns/windows.",
      "[MODEL] Initializing Deep LSTM network in Keras...",
      "[MODEL] LSTM Layer 1 compiled (512 units, returns sequence)",
      "[MODEL] LSTM Layer 2 compiled (512 units, returns sequence)",
      "[MODEL] LSTM Layer 3 compiled (512 units)",
      "[MODEL] FC Bottleneck layer compiled (256 units)",
      "[MODEL] Softmax classification head compiled. Total parameters: 4,192,842.",
      "[SYSTEM] Executing fit operation. Running callbacks..."
    ]);
  };

  // Simulated Training loop logic
  useEffect(() => {
    if (!isTraining) return;

    const delay = simSpeed === "fast" ? 120 : 600;

    const interval = setInterval(() => {
      setSimEpoch(prev => {
        if (prev >= 200) {
          setIsTraining(false);
          setSimLogs(logs => [
            ...logs,
            `\n[SUCCESS] Epoch 200/200 finished! Final loss: 0.124 - validation accuracy: 0.941`,
            `[SYSTEM] Keras best weights saved permanently to 'data/best_music_model.keras'.`,
            `[SYSTEM] Pitch vocabulary vocabulary saved cleanly to 'data/notes_vocabulary.pkl'.`
          ]);
          clearInterval(interval);
          return 200;
        }

        const nextEpoch = prev + 1;
        
        // Decay formulas for loss values and logarithmic rise for accuracies
        const newLoss = Math.max(0.08, 4.8 * Math.pow(0.982, nextEpoch) + (Math.random() * 0.08));
        const newValLoss = Math.max(0.12, 4.9 * Math.pow(0.984, nextEpoch) + (Math.random() * 0.12));
        const newAcc = Math.min(0.98, 0.02 + 0.96 * (1 - Math.pow(0.98, nextEpoch)) + (Math.random() * 0.01));
        const newValAcc = Math.min(0.95, 0.01 + 0.94 * (1 - Math.pow(0.982, nextEpoch)) + (Math.random() * 0.01));

        setSimLoss(Number(newLoss.toFixed(4)));
        setSimValLoss(Number(newValLoss.toFixed(4)));
        setSimAccuracy(Number(newAcc.toFixed(4)));
        setSimValAccuracy(Number(newValAcc.toFixed(4)));

        // Update history for graphs
        setSimHistory(history => [
          ...history,
          {
            epoch: nextEpoch,
            loss: Number(newLoss.toFixed(4)),
            accuracy: Number(newAcc.toFixed(4)),
            valLoss: Number(newValLoss.toFixed(4)),
            valAccuracy: Number(newValAcc.toFixed(4))
          }
        ]);

        // Construct realistic epoch logging lines
        let logLine = `Epoch ${nextEpoch}/200 - loss: ${newLoss.toFixed(4)} - accuracy: ${newAcc.toFixed(3)} - val_loss: ${newValLoss.toFixed(4)} - val_accuracy: ${newValAcc.toFixed(3)}`;
        
        const newLogs = [logLine];

        // Occasional Model Checkpoint outputs
        if (nextEpoch === 1 || nextEpoch % 10 === 0 || nextEpoch === 200) {
          newLogs.unshift(`[CHECKPOINT] Epoch ${nextEpoch}: Loss improved to ${newLoss.toFixed(4)}. Saving model weights...`);
        }

        if (nextEpoch === 50) {
          newLogs.unshift(`[PREVIEW @ Epoch 50] Predictive token samples: C4 -> E4 -> F4 -> G4 -> G#4 (Dissonance factor falling)`);
        } else if (nextEpoch === 100) {
          newLogs.unshift(`[PREVIEW @ Epoch 100] Predictive token samples: A2.E3.A3 -> C4 -> B4 -> E4.G#4.B4 (Functional harmony emerging)`);
        } else if (nextEpoch === 150) {
          newLogs.unshift(`[PREVIEW @ Epoch 150] Predictive token samples: F2.C3 -> F4 -> G4 -> Ab4 -> C5 -> Bb4.Db5 (Complex structures coherent)`);
        }

        setSimLogs(logs => [...logs, ...newLogs]);

        return nextEpoch;
      });
    }, delay);

    return () => clearInterval(interval);
  }, [isTraining, simSpeed, seqLength]);

  // Simulated Custom Note Generation
  const executeNeuralGeneration = () => {
    if (isGenerating) return;
    handleStop();
    setIsGenerating(true);
    
    const steps = [
      "Configuring local deep learning GPU environments...",
      "Initializing predictive token dictionary from 'notes_vocabulary.pkl'...",
      "Extracting a random 100-note starting seed sequence from MIDI corpus...",
      `Processing seed matrix through 3-layer LSTM system (temperature: ${temperature})...`,
      `Synthesizing ${genLength} sequential note vectors using ArgMax sampling...`,
      "Mapping chord normal orders and building output music21 streams...",
      "Generating standard Web Audio sound envelopes..."
    ];

    let currentStepIdx = 0;
    setGenerationStep(steps[0]);

    const interval = setInterval(() => {
      currentStepIdx++;
      if (currentStepIdx >= steps.length) {
        clearInterval(interval);
        
        // Dynamic music generator math
        const generatedNotes = generateCustomSong(generationStyle, temperature, genLength);
        
        // Add new song preset to the list
        const customPreset: SongPreset = {
          id: `custom-gen-${Date.now()}`,
          name: `Custom Generated LSTM Song (${generationStyle.toUpperCase()})`,
          epoch: 200,
          style: `${generationStyle.toUpperCase()} / Generated`,
          description: `An algorithmic 150-note MIDI track generated dynamically on the client side using an emulation of the deep LSTM architecture with temperature set to ${temperature}.`,
          notes: generatedNotes
        };

        setSongsList(prev => [...prev, customPreset]);
        setSelectedSongId(customPreset.id);
        setIsGenerating(false);
        setGenerationStep("");
      } else {
        setGenerationStep(steps[currentStepIdx]);
      }
    }, 450);
  };

  // Export MIDI file trigger
  const triggerMidiDownload = () => {
    // Generate simple standard MIDI data array on the client side or trigger a download placeholder
    const alertMessage = "MIDI File compiled on client side successfully! Downloading 'output_song.mid'.";
    
    // Create a tiny valid dummy MIDI file header and track to physically trigger a real download
    const midiBytes = new Uint8Array([
      0x4d, 0x54, 0x68, 0x64, // MThd
      0x00, 0x00, 0x00, 0x06, // Header size
      0x00, 0x01,             // Format 1
      0x00, 0x01,             // One track
      0x00, 0x60,             // 96 ticks per quarter note
      0x4d, 0x54, 0x72, 0x6b, // MTrk
      0x00, 0x00, 0x00, 0x14, // Chunk size 20
      0x00, 0xff, 0x58, 0x04, 0x04, 0x02, 0x18, 0x08, // Time signature
      0x00, 0xff, 0x51, 0x03, 0x07, 0xa1, 0x20, // Tempo
      0x83, 0x00, 0xff, 0x2f, 0x00 // End of track
    ]);

    const blob = new Blob([midiBytes], { type: "audio/midi" });
    const url = URL.createObjectURL(blob);
    const link = document.createElement("a");
    link.href = url;
    link.download = `lstm_${selectedSongId}.mid`;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    URL.revokeObjectURL(url);
  };

  return (
    <div className="min-h-screen bg-gray-950 text-gray-100 font-sans flex flex-col selection:bg-cyan-500 selection:text-gray-950">
      
      {/* HEADER SECTION */}
      <header className="border-b border-gray-800/80 bg-gray-900/90 backdrop-blur px-4 py-2.5 sticky top-0 z-40 shadow-sm">
        <div className="max-w-7xl mx-auto flex flex-col md:flex-row justify-between items-start md:items-center gap-3">
          
          <div className="flex items-center gap-2">
            <div className="bg-gradient-to-tr from-cyan-500 to-blue-600 p-1.5 rounded-md shadow-sm shadow-cyan-500/10 ring-1 ring-cyan-400/20">
              <Cpu className="w-4 h-4 text-gray-950" />
            </div>
            <div>
              <h1 className="text-base font-bold tracking-tight text-white flex items-center gap-1.5">
                Deep Learning MIDI Music Generator
                <span className="text-[10px] bg-cyan-500/10 text-cyan-400 px-1.5 py-0.5 rounded border border-cyan-500/20 font-mono">
                  LSTM v2.4
                </span>
              </h1>
              <p className="text-[10px] text-gray-400 font-mono leading-none">
                Recurrent Neural Networks for Algorithmic Musical Composition
              </p>
            </div>
          </div>

          <div className="flex items-center gap-2 w-full md:w-auto">
            {/* Audio Waveform Oscilloscope */}
            <div className="bg-gray-950/95 border border-gray-800 rounded-md p-1 flex items-center gap-1.5 w-full md:w-40">
              <span className="text-[9px] font-mono text-gray-500 uppercase tracking-wider pl-1 select-none">
                Synth out
              </span>
              <canvas 
                ref={waveformCanvasRef} 
                width={110} 
                height={20} 
                className="bg-gray-900 rounded-sm border border-gray-800/60 w-full"
              />
            </div>

            {/* Quick Action Info Trigger */}
            <button 
              onClick={() => setShowMLModal(true)}
              className="text-gray-400 hover:text-white bg-gray-850 hover:bg-gray-800 p-1.5 rounded-md transition-colors border border-gray-800 text-xs flex items-center gap-1"
              title="How LSTM Music Generation Works"
            >
              <HelpCircle className="w-4 h-4" />
              <span className="text-[10px] font-mono hidden sm:inline">Info</span>
            </button>
          </div>

        </div>
      </header>

      {/* TABS NAVIGATION BAR */}
      <div className="bg-gray-900/60 border-b border-gray-800 px-4 py-1.5">
        <div className="max-w-7xl mx-auto flex justify-start items-center gap-1.5">
          
          <button
            onClick={() => setActiveTab("composer")}
            className={`flex items-center gap-1.5 px-3 py-1.5 rounded-md font-mono text-[11px] transition-all ${
              activeTab === "composer"
                ? "bg-cyan-500 text-gray-950 font-bold shadow-sm shadow-cyan-500/15"
                : "text-gray-400 hover:text-white hover:bg-gray-800/60"
            }`}
          >
            <Music className="w-3.5 h-3.5" />
            Composer & Player
          </button>

          <button
            onClick={() => setActiveTab("training")}
            className={`flex items-center gap-1.5 px-3 py-1.5 rounded-md font-mono text-[11px] transition-all ${
              activeTab === "training"
                ? "bg-cyan-500 text-gray-950 font-bold shadow-sm shadow-cyan-500/15"
                : "text-gray-400 hover:text-white hover:bg-gray-800/60"
            }`}
          >
            <Activity className="w-3.5 h-3.5" />
            Training Simulator
          </button>

          <button
            onClick={() => setActiveTab("code")}
            className={`flex items-center gap-1.5 px-3 py-1.5 rounded-md font-mono text-[11px] transition-all ${
              activeTab === "code"
                ? "bg-cyan-500 text-gray-950 font-bold shadow-sm shadow-cyan-500/15"
                : "text-gray-400 hover:text-white hover:bg-gray-800/60"
            }`}
          >
            <FileCode className="w-3.5 h-3.5" />
            Python Source Code
          </button>

        </div>
      </div>

      {/* ACTIVE CONTENT DECK */}
      <main className="flex-1 max-w-7xl w-full mx-auto p-4 flex flex-col gap-4">
        
        {/* TAB 1: COMPOSER PLAYER */}
        {activeTab === "composer" && (
          <div className="grid grid-cols-1 lg:grid-cols-12 gap-4 items-start">
            
            {/* Left Control Column */}
            <div className="lg:col-span-4 space-y-4">
              
              {/* Presets & Seeds Deck */}
              <div className="bg-gray-900 border border-gray-800 rounded-md p-3 shadow-md">
                <h2 className="text-xs font-bold font-mono uppercase text-gray-200 tracking-wider mb-2.5 flex items-center gap-1.5">
                  <Sliders className="w-3.5 h-3.5 text-cyan-400" />
                  Select LSTM State Song
                </h2>

                <div className="space-y-2">
                  {songsList.map(song => (
                    <button
                      key={song.id}
                      onClick={() => {
                        handleStop();
                        setSelectedSongId(song.id);
                      }}
                      className={`w-full text-left p-2.5 rounded-md border transition-all flex flex-col gap-0.5 relative overflow-hidden ${
                        selectedSongId === song.id
                          ? "bg-cyan-950/30 border-cyan-500/40 shadow-sm"
                          : "bg-gray-950 hover:bg-gray-850 border-gray-800"
                      }`}
                    >
                      <div className="flex justify-between items-center gap-2">
                        <span className="font-bold text-xs text-white line-clamp-1">
                          {song.name}
                        </span>
                        <span className={`text-[9px] font-mono px-1.5 py-0.5 rounded border uppercase shrink-0 ${
                          song.epoch === 5 
                            ? "bg-red-500/10 text-red-400 border-red-500/20" 
                            : song.epoch === 50 
                            ? "bg-amber-500/10 text-amber-400 border-amber-500/20"
                            : "bg-green-500/10 text-green-400 border-green-500/20"
                        }`}>
                          Ep {song.epoch}
                        </span>
                      </div>
                      
                      <span className="text-[10px] text-cyan-400/90 font-mono">
                        {song.style}
                      </span>
                      
                      <p className="text-[11px] text-gray-400 mt-1 leading-normal">
                        {song.description}
                      </p>
                    </button>
                  ))}
                </div>
              </div>

              {/* Generative Composer Drawer */}
              <div className="bg-gray-900 border border-gray-800 rounded-md p-3.5 shadow-md relative overflow-hidden">
                <div className="absolute top-0 right-0 p-2 opacity-5">
                  <Sparkles className="w-16 h-16 text-cyan-400" />
                </div>

                <h2 className="text-xs font-bold font-mono uppercase text-gray-200 tracking-wider mb-1.5 flex items-center gap-1.5">
                  <Sparkles className="w-3.5 h-3.5 text-cyan-400" />
                  Generative Composition Sandbox
                </h2>
                
                <p className="text-[11px] text-gray-400 leading-normal mb-3">
                  Adjust deep learning hyperparameters to synthesize new compositions locally.
                </p>

                <div className="space-y-3">
                  
                  {/* Style selector */}
                  <div>
                    <label className="block text-[10px] font-mono text-gray-400 mb-1 uppercase tracking-wider">
                      Composition Style Template
                    </label>
                    <select
                      value={generationStyle}
                      onChange={(e) => setGenerationStyle(e.target.value)}
                      className="w-full bg-gray-950 border border-gray-800 rounded px-2 py-1.5 text-xs text-white focus:outline-none focus:border-cyan-500"
                    >
                      <option value="classical">Classical Harmonic (C Major)</option>
                      <option value="synth">Neon Horizon (F Min Pentatonic)</option>
                      <option value="chiptune">Retro Arcade (Blues / Fast)</option>
                      <option value="bach">Stretched Fugue (Harmonic Minor)</option>
                    </select>
                  </div>

                  {/* Temperature slider */}
                  <div>
                    <div className="flex justify-between items-center mb-0.5">
                      <label className="text-[10px] font-mono text-gray-400 uppercase tracking-wider">
                        Temperature (Randomness)
                      </label>
                      <span className="text-xs font-mono text-cyan-400 font-bold">
                        {temperature.toFixed(2)}
                      </span>
                    </div>
                    <input
                      type="range"
                      min={0.2}
                      max={1.5}
                      step={0.05}
                      value={temperature}
                      onChange={(e) => setTemperature(parseFloat(e.target.value))}
                      className="w-full accent-cyan-500 h-1"
                    />
                    <div className="flex justify-between text-[8px] text-gray-500 font-mono mt-0.5">
                      <span>Strict</span>
                      <span>Balanced</span>
                      <span>Creative</span>
                    </div>
                  </div>

                  {/* Generation length slider */}
                  <div>
                    <div className="flex justify-between items-center mb-0.5">
                      <label className="text-[10px] font-mono text-gray-400 uppercase tracking-wider">
                        Generation Note Length
                      </label>
                      <span className="text-xs font-mono text-cyan-400 font-bold">
                        {genLength} notes
                      </span>
                    </div>
                    <input
                      type="range"
                      min={50}
                      max={300}
                      step={25}
                      value={genLength}
                      onChange={(e) => setGenLength(parseInt(e.target.value, 10))}
                      className="w-full accent-cyan-500 h-1"
                    />
                  </div>

                  {/* Generation Trigger button */}
                  <button
                    onClick={executeNeuralGeneration}
                    disabled={isGenerating}
                    className="w-full bg-gradient-to-r from-cyan-500 to-blue-600 hover:from-cyan-400 hover:to-blue-500 text-gray-950 font-bold text-xs font-mono py-2 rounded transition-all shadow-sm active:scale-[0.99] disabled:opacity-50 disabled:cursor-not-allowed flex justify-center items-center gap-1.5"
                  >
                    {isGenerating ? (
                      <>
                        <span className="animate-spin text-xs">⌛</span>
                        Generating...
                      </>
                    ) : (
                      <>
                        <Sparkles className="w-3.5 h-3.5 shrink-0" />
                        Run Generator
                      </>
                    )}
                  </button>

                  {/* Loading generation block overlay */}
                  {isGenerating && (
                    <div className="bg-gray-950/90 border border-cyan-500/20 rounded p-2.5 text-center space-y-1.5 mt-1.5">
                      <div className="flex justify-center">
                        <div className="animate-bounce h-1.5 w-1.5 bg-cyan-500 rounded-full mx-0.5"></div>
                        <div className="animate-bounce h-1.5 w-1.5 bg-cyan-500 rounded-full mx-0.5 [animation-delay:0.2s]"></div>
                        <div className="animate-bounce h-1.5 w-1.5 bg-cyan-500 rounded-full mx-0.5 [animation-delay:0.4s]"></div>
                      </div>
                      <p className="text-[10px] font-mono text-cyan-400 animate-pulse">
                        {generationStep}
                      </p>
                    </div>
                  )}

                </div>
              </div>

            </div>            {/* Right Piano Roll & Synthesizer Main Column */}
            <div className="lg:col-span-8 space-y-4">
              
              {/* Audio Controls Console Bar */}
              <div className="bg-gray-900 border border-gray-800 rounded-md p-2.5 shadow-md flex flex-wrap justify-between items-center gap-3">
                
                {/* Synthesis controls: play, stop */}
                <div className="flex items-center gap-1.5">
                  <button
                    onClick={handlePlayPause}
                    className={`p-2.5 rounded-md flex items-center justify-center transition-all ${
                      isPlaying 
                        ? "bg-amber-500 hover:bg-amber-400 text-gray-950 shadow-sm" 
                        : "bg-cyan-500 hover:bg-cyan-400 text-gray-950 shadow-sm"
                    }`}
                    title={isPlaying ? "Pause Composition" : "Play Composition"}
                  >
                    {isPlaying ? <Pause className="w-4 h-4 fill-current" /> : <Play className="w-4 h-4 fill-current" />}
                  </button>

                  <button
                    onClick={handleStop}
                    className="p-2.5 bg-gray-800 hover:bg-gray-700 text-gray-300 hover:text-white rounded-md transition-colors border border-gray-700"
                    title="Stop & Reset Timeline"
                  >
                    <Square className="w-4 h-4 fill-current" />
                  </button>

                  <button
                    onClick={handleRestart}
                    className="p-2.5 bg-gray-800 hover:bg-gray-700 text-gray-300 hover:text-white rounded-md transition-colors border border-gray-700"
                    title="Restart from Beginning"
                  >
                    <RotateCcw className="w-4 h-4" />
                  </button>
                </div>

                {/* Instrument Wave / Playback variables selector */}
                <div className="flex items-center gap-2 flex-wrap">
                  
                  {/* Wave Instrument Type Selector */}
                  <div className="flex items-center gap-1.5 bg-gray-950 px-2 py-1 rounded-md border border-gray-800">
                    <span className="text-[9px] font-mono text-gray-500 uppercase select-none">
                      Wave
                    </span>
                    <select
                      value={instrumentType}
                      onChange={(e) => setInstrumentType(e.target.value as any)}
                      className="bg-transparent text-xs text-white focus:outline-none font-bold"
                    >
                      <option value="piano">Sine (Piano)</option>
                      <option value="synth">Sawtooth (Synth)</option>
                      <option value="organ">Triangle (Organ)</option>
                    </select>
                  </div>

                  {/* BPM Tempo slider */}
                  <div className="flex items-center gap-1.5 bg-gray-950 px-2 py-1 rounded-md border border-gray-800">
                    <span className="text-[9px] font-mono text-gray-500 uppercase select-none">
                      Tempo
                    </span>
                    <input
                      type="range"
                      min={60}
                      max={180}
                      step={5}
                      value={tempo}
                      onChange={(e) => setTempo(parseInt(e.target.value, 10))}
                      className="w-12 accent-cyan-500 h-1"
                    />
                    <span className="text-xs font-mono font-bold text-white w-12 text-right">
                      {tempo} BPM
                    </span>
                  </div>

                  {/* Master Volume */}
                  <div className="flex items-center gap-1.5 bg-gray-950 px-2 py-1 rounded-md border border-gray-800">
                    <Volume2 className="w-3.5 h-3.5 text-gray-500" />
                    <input
                      type="range"
                      min={0}
                      max={1}
                      step={0.1}
                      value={volume}
                      onChange={(e) => setVolume(parseFloat(e.target.value))}
                      className="w-12 accent-cyan-500 h-1"
                    />
                    <span className="text-xs font-mono font-bold text-white w-7 text-right">
                      {Math.round(volume * 100)}%
                    </span>
                  </div>

                </div>

                {/* MIDI Exporter */}
                <button
                  onClick={triggerMidiDownload}
                  className="bg-gray-800 hover:bg-gray-700 border border-gray-700 hover:border-gray-650 px-2.5 py-1.5 rounded-md text-xs font-mono font-bold text-cyan-400 hover:text-cyan-300 transition-colors flex items-center gap-1 ml-auto md:ml-0"
                >
                  <Download className="w-3.5 h-3.5 shrink-0" />
                  MIDI
                </button>

              </div>

              {/* Main Piano Roll Visualizer Window */}
              <div className="bg-gray-900 border border-gray-800 rounded-md overflow-hidden shadow-md">
                
                {/* Visualizer header metrics info bar */}
                <div className="bg-gray-950 border-b border-gray-800/80 px-3 py-2 flex justify-between items-center">
                  <div className="flex items-center gap-1.5">
                    <div className="h-1.5 w-1.5 bg-green-500 rounded-full animate-ping"></div>
                    <span className="text-[11px] font-mono font-bold text-white uppercase tracking-wider">
                      Live Scrolling Piano Roll Matrix
                    </span>
                  </div>
                  
                  <div className="flex items-center gap-3 text-[10px] font-mono text-gray-400">
                    <div>
                      Active: <span className="text-cyan-400 font-bold">{activeNotes.size}</span>
                    </div>
                    <div>
                      Beat: <span className="text-pink-400 font-bold">{playbackTime.toFixed(2)}</span>
                    </div>
                  </div>
                </div>

                {/* Canvas roll viewport */}
                <div className="relative bg-gray-950 border-b border-gray-800">
                  <canvas 
                    ref={pianoRollCanvasRef}
                    width={800}
                    height={260}
                    className="w-full h-64 block cursor-default"
                  />
                  
                  {/* Floating help cue */}
                  <div className="absolute bottom-2 right-2 bg-gray-900/95 border border-gray-800 px-2.5 py-1 rounded pointer-events-none flex items-center gap-1.5 max-w-xs">
                    <span className="text-[9px] font-mono text-gray-400 leading-normal">
                      Pink playhead line is stationary; notes scroll from right to left.
                    </span>
                  </div>
                </div>

                {/* Lower Virtual MIDI Keyboard Visualizer */}
                <div className="bg-gray-900 p-3 border-t border-gray-800">
                  <div className="flex justify-between items-center mb-1.5">
                    <span className="text-[9px] font-mono text-gray-400 uppercase tracking-widest">
                      Octaves: C3 - B4 (Interactive Synthesizer Keyboard)
                    </span>
                    <span className="text-[9px] font-mono text-gray-500 italic">
                      Click keys to play manually
                    </span>
                  </div>

                  <div className="flex relative select-none w-full max-w-2xl mx-auto h-16 border border-gray-800 rounded-sm overflow-hidden">
                    {/* Build out white and black piano keys */}
                    {Array.from({ length: 24 }, (_, i) => {
                      const midiNum = 48 + i; // C3 to B4
                      const semitone = midiNum % 12;
                      const isBlackKey = [1, 3, 6, 8, 10].includes(semitone);
                      const isLit = activeNotes.has(midiNum);

                      const keyLabels = ["C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A", "A#", "B"];
                      const octaveLabel = Math.floor(midiNum / 12) - 1;
                      const label = `${keyLabels[semitone]}${octaveLabel}`;

                      const triggerManualNote = () => {
                        initAudio();
                        playSynthesizerNotes([noteToFreqs(`${midiNum}`)[0]], 0.5);
                        // Briefly light key
                        setActiveNotes(new Set([midiNum]));
                        setTimeout(() => setActiveNotes(new Set()), 300);
                      };

                      return (
                        <button
                          key={midiNum}
                          onClick={triggerManualNote}
                          className={`flex-1 transition-all flex flex-col justify-end items-center pb-1.5 select-none relative ${
                            isBlackKey
                              ? "bg-gray-950 text-gray-500 hover:bg-gray-800 border-l border-r border-gray-900 z-10 -mx-2 h-10 h-3/5"
                              : "bg-white text-gray-400 hover:bg-gray-100 border-r border-gray-200 z-0 h-full"
                          } ${
                            isLit 
                              ? isBlackKey 
                                ? "bg-cyan-500! text-gray-950!" 
                                : "bg-cyan-400! text-gray-950! shadow-inner"
                              : ""
                          }`}
                        >
                          <span className={`text-[8px] font-mono uppercase tracking-tighter block pointer-events-none select-none ${
                            isBlackKey ? "text-[5px]" : "text-gray-500 font-bold"
                          }`}>
                            {label}
                          </span>
                        </button>
                      );
                    })}
                  </div>
                </div>

              </div>

            </div>

          </div>
        )}

        {/* TAB 2: TRAINING SIMULATOR */}
        {activeTab === "training" && (
          <div className="space-y-4">
            
            {/* Simulation Dashboard Deck */}
            <div className="grid grid-cols-1 lg:grid-cols-12 gap-4">
              
              {/* Left Settings Sidebar */}
              <div className="lg:col-span-4 bg-gray-900 border border-gray-800 rounded-md p-3.5 shadow-md space-y-4">
                <h2 className="text-xs font-bold font-mono uppercase text-gray-200 tracking-wider flex items-center gap-1.5">
                  <Sliders className="w-3.5 h-3.5 text-cyan-400" />
                  Keras Train Config
                </h2>

                {/* Form fields */}
                <div className="space-y-3.5 text-xs font-mono">
                  
                  <div>
                    <label className="block text-gray-400 mb-0.5 text-[10px] tracking-wider">DATASET SONGS DIRECTORY</label>
                    <input 
                      type="text" 
                      value="midi_songs/" 
                      disabled 
                      className="w-full bg-gray-950 border border-gray-800 rounded px-2.5 py-1.5 text-gray-400 cursor-not-allowed"
                    />
                  </div>

                  <div>
                    <label className="block text-gray-400 mb-0.5 text-[10px] tracking-wider">SEQUENCE LENGTH (X_WINDOW)</label>
                    <select
                      value={seqLength}
                      onChange={(e) => setSeqLength(parseInt(e.target.value, 10))}
                      disabled={isTraining}
                      className="w-full bg-gray-950 border border-gray-800 rounded px-2.5 py-1.5 text-white focus:outline-none focus:border-cyan-500 disabled:opacity-50"
                    >
                      <option value={50}>50 note vectors</option>
                      <option value={100}>100 note vectors (Recommended)</option>
                      <option value={150}>150 note vectors</option>
                    </select>
                  </div>

                  <div>
                    <label className="block text-gray-400 mb-0.5 text-[10px] tracking-wider">TOTAL TRAINING EPOCHS</label>
                    <input 
                      type="number" 
                      value={200} 
                      disabled 
                      className="w-full bg-gray-950 border border-gray-800 rounded px-2.5 py-1.5 text-gray-400 cursor-not-allowed"
                    />
                  </div>

                  <div>
                    <label className="block text-gray-400 mb-0.5 text-[10px] tracking-wider">OPTIMIZER ALGORITHM</label>
                    <input 
                      type="text" 
                      value="RMSprop (learning_rate=0.001)" 
                      disabled 
                      className="w-full bg-gray-950 border border-gray-800 rounded px-2.5 py-1.5 text-gray-400 cursor-not-allowed"
                    />
                  </div>

                  <div>
                    <label className="block text-gray-400 mb-0.5 text-[10px] tracking-wider">LOSS COMPILATION</label>
                    <input 
                      type="text" 
                      value="categorical_crossentropy" 
                      disabled 
                      className="w-full bg-gray-950 border border-gray-800 rounded px-2.5 py-1.5 text-gray-400 cursor-not-allowed"
                    />
                  </div>

                  <div>
                    <label className="block text-gray-400 mb-0.5 text-[10px] tracking-wider">SIMULATION SPEED RATE</label>
                    <div className="grid grid-cols-2 gap-1.5 mt-0.5">
                      <button
                        onClick={() => setSimSpeed("normal")}
                        className={`py-1 px-2 rounded border text-center font-bold text-[10px] ${
                          simSpeed === "normal"
                            ? "bg-cyan-950 border-cyan-500 text-cyan-400"
                            : "bg-gray-950 border-gray-800 text-gray-400 hover:text-white"
                        }`}
                      >
                        Standard (Realtime)
                      </button>
                      <button
                        onClick={() => setSimSpeed("fast")}
                        className={`py-1 px-2 rounded border text-center font-bold text-[10px] ${
                          simSpeed === "fast"
                            ? "bg-cyan-950 border-cyan-500 text-cyan-400"
                            : "bg-gray-950 border-gray-800 text-gray-400 hover:text-white"
                        }`}
                      >
                        Fast (Hyperdrive)
                      </button>
                    </div>
                  </div>

                  {/* Trigger training */}
                  <button
                    onClick={startSimulatedTraining}
                    className={`w-full py-2 rounded text-xs font-bold text-center transition-all flex items-center justify-center gap-1.5 ${
                      isTraining
                        ? "bg-red-500/10 text-red-400 border border-red-500/30 hover:bg-red-500/20"
                        : "bg-cyan-500 text-gray-950 font-bold hover:bg-cyan-400 shadow-sm"
                    }`}
                  >
                    <Cpu className="w-3.5 h-3.5" />
                    {isTraining ? "Halt Training Session" : "Start Training Session"}
                  </button>

                </div>
              </div>

              {/* Right Charts and Dashboard Outputs */}
              <div className="lg:col-span-8 bg-gray-900 border border-gray-800 rounded-md p-3.5 shadow-md flex flex-col gap-4">
                
                {/* Real-time Metric Badges */}
                <div className="grid grid-cols-2 md:grid-cols-4 gap-2.5">
                  
                  <div className="bg-gray-950 border border-gray-800 p-2.5 rounded-md text-center space-y-0.5">
                    <span className="text-[9px] font-mono text-gray-500 uppercase tracking-wider block">
                      Active Epoch
                    </span>
                    <span className="text-xl font-bold font-mono text-white">
                      {simEpoch}/200
                    </span>
                  </div>

                  <div className="bg-gray-950 border border-gray-800 p-2.5 rounded-md text-center space-y-0.5">
                    <span className="text-[9px] font-mono text-gray-500 uppercase tracking-wider block">
                      Loss (CCE)
                    </span>
                    <span className="text-xl font-bold font-mono text-cyan-400">
                      {simLoss.toFixed(3)}
                    </span>
                  </div>

                  <div className="bg-gray-950 border border-gray-800 p-2.5 rounded-md text-center space-y-0.5">
                    <span className="text-[9px] font-mono text-gray-500 uppercase tracking-wider block">
                      Accuracy
                    </span>
                    <span className="text-xl font-bold font-mono text-green-400">
                      {Math.round(simAccuracy * 100)}%
                    </span>
                  </div>

                  <div className="bg-gray-950 border border-gray-800 p-2.5 rounded-md text-center space-y-0.5">
                    <span className="text-[9px] font-mono text-gray-500 uppercase tracking-wider block">
                      Val Loss
                    </span>
                    <span className="text-xl font-bold font-mono text-purple-400">
                      {simValLoss.toFixed(3)}
                    </span>
                  </div>

                </div>

                {/* Mini Metric SVG Lines Charts */}
                <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                  
                  {/* Loss Chart */}
                  <div className="bg-gray-950 border border-gray-800 p-2.5 rounded-md space-y-1.5">
                    <div className="flex justify-between items-center">
                      <h3 className="text-xs font-mono font-bold text-gray-300 uppercase flex items-center gap-1">
                        <TrendingDown className="w-3 h-3 text-cyan-400" />
                        Loss Curve Decay
                      </h3>
                      <span className="text-[10px] font-mono text-cyan-400 font-bold">
                        {simLoss.toFixed(4)}
                      </span>
                    </div>

                    <div className="h-28 bg-gray-900 border border-gray-800/60 rounded p-0.5 flex items-end relative">
                      {/* Grid markers */}
                      <div className="absolute left-1.5 top-1.5 text-[8px] font-mono text-gray-600">Max: 4.9</div>
                      <div className="absolute left-1.5 bottom-1.5 text-[8px] font-mono text-gray-600">Min: 0.1</div>

                      {/* SVG Line representation of loss decay */}
                      {simHistory.length > 1 ? (
                        <svg className="w-full h-full" viewBox="0 0 200 100" preserveAspectRatio="none">
                          {/* Loss curve */}
                          <path
                            d={simHistory.reduce((acc, point, idx) => {
                              const x = (idx / simHistory.length) * 200;
                              const y = 100 - (point.loss / 5.0) * 100;
                              return `${acc} ${idx === 0 ? "M" : "L"} ${x} ${y}`;
                            }, "")}
                            fill="none"
                            stroke="#06b6d4"
                            strokeWidth="2"
                          />
                          {/* Val loss curve */}
                          <path
                            d={simHistory.reduce((acc, point, idx) => {
                              const x = (idx / simHistory.length) * 200;
                              const y = 100 - (point.valLoss / 5.0) * 100;
                              return `${acc} ${idx === 0 ? "M" : "L"} ${x} ${y}`;
                            }, "")}
                            fill="none"
                            stroke="#a855f7"
                            strokeWidth="1.5"
                            strokeDasharray="3"
                          />
                        </svg>
                      ) : (
                        <span className="text-[10px] font-mono text-gray-500 w-full text-center py-8 select-none">
                          Awaiting training triggers...
                        </span>
                      )}
                    </div>
                    <div className="flex justify-between text-[8px] text-gray-650 font-mono">
                      <span>Epoch 0</span>
                      <span>Epoch 200</span>
                    </div>
                  </div>

                  {/* Accuracy Chart */}
                  <div className="bg-gray-950 border border-gray-800 p-2.5 rounded-md space-y-1.5">
                    <div className="flex justify-between items-center">
                      <h3 className="text-xs font-mono font-bold text-gray-300 uppercase flex items-center gap-1">
                        <Activity className="w-3 h-3 text-green-400" />
                        Accuracy Target Rise
                      </h3>
                      <span className="text-[10px] font-mono text-green-400 font-bold">
                        {(simAccuracy * 100).toFixed(1)}%
                      </span>
                    </div>

                    <div className="h-28 bg-gray-900 border border-gray-800/60 rounded p-0.5 flex items-end relative">
                      <div className="absolute left-1.5 top-1.5 text-[8px] font-mono text-gray-600">Max: 100%</div>
                      <div className="absolute left-1.5 bottom-1.5 text-[8px] font-mono text-gray-600">Min: 0%</div>

                      {simHistory.length > 1 ? (
                        <svg className="w-full h-full" viewBox="0 0 200 100" preserveAspectRatio="none">
                          {/* Accuracy path */}
                          <path
                            d={simHistory.reduce((acc, point, idx) => {
                              const x = (idx / simHistory.length) * 200;
                              const y = 100 - point.accuracy * 100;
                              return `${acc} ${idx === 0 ? "M" : "L"} ${x} ${y}`;
                            }, "")}
                            fill="none"
                            stroke="#22c55e"
                            strokeWidth="2"
                          />
                          {/* Val accuracy path */}
                          <path
                            d={simHistory.reduce((acc, point, idx) => {
                              const x = (idx / simHistory.length) * 200;
                              const y = 100 - point.valAccuracy * 100;
                              return `${acc} ${idx === 0 ? "M" : "L"} ${x} ${y}`;
                            }, "")}
                            fill="none"
                            stroke="#a855f7"
                            strokeWidth="1.5"
                            strokeDasharray="3"
                          />
                        </svg>
                      ) : (
                        <span className="text-[10px] font-mono text-gray-500 w-full text-center py-8 select-none">
                          Awaiting training triggers...
                        </span>
                      )}
                    </div>
                    <div className="flex justify-between text-[8px] text-gray-650 font-mono">
                      <span>Epoch 0</span>
                      <span>Epoch 200</span>
                    </div>
                  </div>

                </div>

              </div>

            </div>

            {/* Immersive Terminal Output */}
            <div className="bg-gray-900 border border-gray-800 rounded-md overflow-hidden shadow-md flex flex-col">
              
              <div className="bg-gray-950 px-3 py-2 border-b border-gray-800 flex justify-between items-center">
                <div className="flex items-center gap-1.5">
                  <Terminal className="w-3.5 h-3.5 text-cyan-400" />
                  <span className="text-[11px] font-mono text-white font-bold uppercase tracking-wider">
                    Interactive Python stdout/stderr Logs
                  </span>
                </div>
                
                {isTraining && (
                  <span className="text-[9px] font-mono text-cyan-400 bg-cyan-950/50 border border-cyan-500/20 px-1.5 py-0.5 rounded animate-pulse">
                    ACTIVE COMPILING TENSORS
                  </span>
                )}
              </div>

              {/* Console log wrapper */}
              <div className="p-3 bg-gray-950 text-[11px] font-mono text-gray-300 h-52 overflow-y-auto space-y-0.5 scrollbar-thin scrollbar-thumb-gray-800">
                {simLogs.length === 0 ? (
                  <p className="text-gray-500 italic">Console is quiet. Click 'Start Training Session' above to initiate LSTM compiling routines.</p>
                ) : (
                  simLogs.map((log, i) => (
                    <div 
                      key={i} 
                      className={`leading-relaxed ${
                        log.startsWith("[SYSTEM]") 
                          ? "text-blue-400" 
                          : log.startsWith("[MODEL]") 
                          ? "text-purple-400" 
                          : log.startsWith("[CHECKPOINT]") 
                          ? "text-yellow-500 font-bold"
                          : log.startsWith("[SUCCESS]")
                          ? "text-green-400 font-bold"
                          : log.startsWith("[PREVIEW")
                          ? "text-cyan-400 italic pl-3"
                          : "text-gray-300"
                      }`}
                    >
                      {log}
                    </div>
                  ))
                )}
              </div>

            </div>

          </div>
        )}

        {/* TAB 3: COMPLETE PYTHON CODE VIEWER */}
        {activeTab === "code" && (
          <div className="space-y-4">
            
            {/* Model Architecture Flow Card */}
            <div className="bg-gray-900 border border-gray-800 rounded-md p-3.5 shadow-md">
              <h2 className="text-xs font-bold font-mono uppercase text-gray-200 tracking-wider mb-2.5 flex items-center gap-1.5">
                <Cpu className="w-3.5 h-3.5 text-cyan-400" />
                Network Architecture Layout Map
              </h2>

              <p className="text-[11px] text-gray-400 leading-normal mb-4">
                Below is a precise representation of the matrix shapes and tensor transformations that occur at each layer of the deep LSTM music model requested.
              </p>

              {/* Graphical representation of the layers */}
              <div className="grid grid-cols-1 md:grid-cols-5 gap-2.5 items-center">
                
                <div className="bg-gray-950 border border-gray-800 p-2.5 rounded-md text-center flex flex-col justify-center items-center h-24 relative">
                  <span className="text-[9px] font-mono text-cyan-400 uppercase font-bold mb-0.5">Input Matrix</span>
                  <p className="text-xs text-white">Window of Notes</p>
                  <p className="text-[9px] font-mono text-gray-500 mt-1">Shape: (N, 100, 1)</p>
                  <div className="hidden md:block absolute -right-2 top-1/2 -translate-y-1/2 text-gray-700 text-sm">➜</div>
                </div>

                <div className="bg-gray-950 border border-gray-800 p-2.5 rounded-md text-center flex flex-col justify-center items-c
