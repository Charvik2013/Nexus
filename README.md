import React, { useState, useEffect } from 'react';
import { ViewState, PlayerState, Subject, Avatar, Artifact } from './types';
import { generateWorksheet, generateArtifact } from './services/geminiService';
import QuizArena from './components/QuizArena';
import ProfileVault from './components/ProfileVault';
import MiniGames from './components/MiniGames';
import { Star, BookOpen, User, Search, Calculator, FlaskConical, Languages, Globe, Code, GraduationCap, LayoutGrid, Gamepad2 } from 'lucide-react';

// --- Constants ---
const SUBJECTS: Subject[] = [
  { 
    id: 'math', 
    name: 'Mathematics', 
    topicKeyword: 'Math',
    description: 'Numbers, Logic, and Problem Solving.', 
    icon: 'calculator',
    color: 'text-blue-400', 
  },
  { 
    id: 'science', 
    name: 'Science', 
    topicKeyword: 'Science',
    description: 'Biology, Chemistry, and Physics.', 
    icon: 'flask',
    color: 'text-green-400',
  },
  { 
    id: 'english', 
    name: 'English', 
    topicKeyword: 'English Language Arts',
    description: 'Grammar, Reading Comprehension, and Vocabulary.', 
    icon: 'languages',
    color: 'text-yellow-400',
  },
  { 
    id: 'history', 
    name: 'History', 
    topicKeyword: 'History',
    description: 'Past events, Civilizations, and Cultures.', 
    icon: 'globe',
    color: 'text-orange-400',
  },
  {
    id: 'coding',
    name: 'Coding',
    topicKeyword: 'Computer Science',
    description: 'Logic, Algorithms, and Digital Skills.',
    icon: 'code',
    color: 'text-purple-400'
  }
];

const AVATARS: Avatar[] = [
  { id: 'student', name: 'Student', icon: 'user', color: 'text-space-light', requiredLevel: 1 },
  { id: 'scholar', name: 'Scholar', icon: 'box', color: 'text-blue-400', requiredLevel: 3 },
  { id: 'graduate', name: 'Graduate', icon: 'star', color: 'text-cyan-400', requiredLevel: 5 },
  { id: 'professor', name: 'Professor', icon: 'shield', color: 'text-green-400', requiredLevel: 10 },
  { id: 'genius', name: 'Genius', icon: 'zap', color: 'text-indigo-400', requiredLevel: 20 },
];

const INITIAL_PLAYER_STATE: PlayerState = {
  xp: 0,
  grade: 0, // 0 means not selected yet
  level: 1,
  completedWorksheets: 0,
  recentScores: [],
  inventory: [],
  unlockedAvatars: ['student'],
  currentAvatar: 'student'
};

// --- Main App Component ---

const App: React.FC = () => {
  const [view, setView] = useState<ViewState>(ViewState.DASHBOARD);
  const [player, setPlayer] = useState<PlayerState>(INITIAL_PLAYER_STATE);
  const [activeSubject, setActiveSubject] = useState<Subject | null>(null);
  const [quizData, setQuizData] = useState<any>(null);
  const [loading, setLoading] = useState(false);
  const [customTopic, setCustomTopic] = useState("");

  // Load state
  useEffect(() => {
    const saved = localStorage.getItem('nexus_school_v1');
    if (saved) {
      const parsed = JSON.parse(saved);
      setPlayer({ ...INITIAL_PLAYER_STATE, ...parsed });
      // If user hasn't selected grade yet, force that view
      if (!parsed.grade || parsed.grade === 0) {
        setView(ViewState.GRADE_SELECT);
      }
    } else {
      setView(ViewState.GRADE_SELECT);
    }
  }, []);

  // Save state
  useEffect(() => {
    localStorage.setItem('nexus_school_v1', JSON.stringify(player));
    
    // Check for Avatar Unlocks
    let newAvatars = [...player.unlockedAvatars];
    AVATARS.forEach(avatar => {
        if (player.level >= avatar.requiredLevel && !player.unlockedAvatars.includes(avatar.id)) {
            newAvatars.push(avatar.id);
        }
    });

    if (newAvatars.length > player.unlockedAvatars.length) {
        setPlayer(prev => ({ ...prev, unlockedAvatars: newAvatars }));
    }

  }, [player.xp, player.level, player.grade]);

  const handleGradeSelect = (grade: number) => {
    setPlayer(prev => ({ ...prev, grade }));
    setView(ViewState.DASHBOARD);
  };

  const handleSubjectSelect = async (subject: Subject) => {
    setActiveSubject(subject);
    startWorksheet(subject.topicKeyword);
  };

  const handleCustomSubject = () => {
    if (!customTopic.trim()) return;
    const tempSubject: Subject = {
      id: 'custom',
      name: customTopic,
      topicKeyword: customTopic,
      description: 'Custom Study Module',
      icon: 'search',
      color: 'text-white'
    };
    setActiveSubject(tempSubject);
    startWorksheet(customTopic);
  };

  const startWorksheet = async (topic: string) => {
    setLoading(true);
    // Always generate 5 questions for a worksheet
    const data = await generateWorksheet(topic, player.grade, 5);
    setQuizData(data);
    setView(ViewState.QUIZ);
    setLoading(false);
  };

  const handleWorksheetComplete = async (score: number, totalQuestions: number) => {
    const percentage = score / totalQuestions;
    const xpEarned = (score * 20) + (totalQuestions * 5); 
    
    const newHistory = [...player.recentScores, percentage].slice(-5);
    
    // Chance for artifact if score is perfect
    let newArtifact: Artifact | null = null;
    if (percentage === 1.0 && activeSubject) {
        const artifactData = await generateArtifact(activeSubject.topicKeyword);
        if (artifactData && artifactData.name) {
            newArtifact = {
                id: Date.now().toString(),
                name: artifactData.name || "Sticker",
                description: artifactData.description || "A reward for excellence.",
                rarity: (artifactData.rarity as any) || 'Common',
                dateAcquired: Date.now()
            };
        }
    }

    setPlayer(prev => ({
      ...prev,
      xp: prev.xp + xpEarned,
      completedWorksheets: prev.completedWorksheets + 1,
      level: Math.floor((prev.xp + xpEarned) / 300) + 1,
      recentScores: newHistory,
      inventory: newArtifact ? [...prev.inventory, newArtifact] : prev.inventory
    }));
    
    // If passed (>50%), go to Arcade
    if (percentage >= 0.5) {
        setView(ViewState.ARCADE);
    } else {
        setView(ViewState.DASHBOARD);
    }
  };

  const handleAvatarSelect = (id: string) => {
    setPlayer(prev => ({ ...prev, currentAvatar: id }));
  };

  const renderGradeSelector = () => (
    <div className="flex flex-col items-center justify-center min-h-[80vh] px-4 animate-in fade-in slide-in-from-bottom-8">
      <GraduationCap className="w-20 h-20 text-space-accent mb-6" />
      <h1 className="text-4xl md:text-5xl font-black text-white mb-4 text-center">Student Profile</h1>
      <p className="text-space-light text-xl mb-10 text-center">Select your current Grade Level to initialize the curriculum.</p>
      
      <div className="grid grid-cols-3 sm:grid-cols-4 md:grid-cols-6 gap-4 max-w-3xl w-full">
        {Array.from({ length: 12 }, (_, i) => i + 1).map((grade) => (
          <button
            key={grade}
            onClick={() => handleGradeSelect(grade)}
            className="p-6 rounded-xl bg-space-800 border-2 border-space-700 hover:border-space-accent hover:bg-space-700 hover:scale-105 transition-all flex flex-col items-center gap-2 group"
          >
            <span className="text-sm text-space-light group-hover:text-white uppercase font-bold">Grade</span>
            <span className="text-3xl font-black text-white group-hover:text-cyan-300">{grade}</span>
          </button>
        ))}
      </div>
    </div>
  );

  const renderDashboard = () => (
    <div className="max-w-5xl mx-auto py-10 px-4">
      <header className="mb-12 text-center">
        <h1 className="text-4xl md:text-6xl font-black text-transparent bg-clip-text bg-gradient-to-r from-cyan-400 to-green-400 mb-4 tracking-tight">
          NEXUS LEARNING
        </h1>
        <div className="inline-flex items-center gap-2 px-4 py-2 bg-space-800 rounded-full border border-space-700 text-space-light">
          <GraduationCap size={18} />
          <span className="font-bold">Grade {player.grade} Curriculum</span>
          <button onClick={() => setView(ViewState.GRADE_SELECT)} className="ml-2 text-xs text-space-accent hover:underline">Change</button>
        </div>
      </header>

      {/* Search Bar */}
      <div className="mb-12 max-w-xl mx-auto">
        <div className="relative flex items-center bg-space-900 rounded-xl p-2 border border-space-700 focus-within:border-space-accent transition-colors">
            <Search className="ml-3 text-space-light w-5 h-5" />
            <input 
                type="text" 
                value={customTopic}
                onChange={(e) => setCustomTopic(e.target.value)}
                placeholder="Study any topic (e.g. 'Dinosaurs', 'Fractions')"
                className="w-full bg-transparent text-white px-4 py-3 focus:outline-none placeholder-space-700"
                onKeyDown={(e) => e.key === 'Enter' && handleCustomSubject()}
            />
            <button 
                onClick={handleCustomSubject}
                className="bg-space-accent hover:bg-cyan-400 text-space-900 px-6 py-2 rounded-lg font-bold transition-transform active:scale-95"
            >
                Start
            </button>
        </div>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        {SUBJECTS.map((subject) => (
          <button
            key={subject.id}
            onClick={() => handleSubjectSelect(subject)}
            className="relative group flex flex-col items-start text-left p-6 rounded-2xl border-2 border-space-700 bg-space-900/40 hover:bg-space-800 hover:border-space-accent transition-all duration-300 hover:shadow-[0_0_20px_rgba(34,211,238,0.15)]"
          >
            <div className={`p-3 rounded-xl mb-4 bg-space-800 group-hover:bg-space-700 transition-colors ${subject.color}`}>
              {subject.icon === 'calculator' && <Calculator size={28} />}
              {subject.icon === 'flask' && <FlaskConical size={28} />}
              {subject.icon === 'languages' && <Languages size={28} />}
              {subject.icon === 'globe' && <Globe size={28} />}
              {subject.icon === 'code' && <Code size={28} />}
            </div>
            
            <h3 className={`text-2xl font-bold mb-2 text-white group-hover:text-cyan-300`}>{subject.name}</h3>
            <p className="text-sm text-space-light mb-6">{subject.description}</p>
            
            <div className="mt-auto flex items-center gap-2 text-xs font-bold text-space-accent uppercase tracking-wider">
               <BookOpen size={14} />
               <span>Start Worksheet</span>
            </div>
          </button>
        ))}
      </div>
    </div>
  );

  return (
    <div className="min-h-screen text-white font-sans selection:bg-space-accent selection:text-space-900 overflow-x-hidden">
      
      {/* Top Navigation / Stats Bar (Only show if not in Grade Select) */}
      {view !== ViewState.GRADE_SELECT && (
        <div className="sticky top-0 z-50 glass-panel border-b-0 border-b-space-700/30">
            <div className="max-w-6xl mx-auto px-4 h-16 flex items-center justify-between">
                <div 
                    className="flex items-center gap-2 font-bold text-xl tracking-tight cursor-pointer hover:opacity-80 transition-opacity"
                    onClick={() => setView(ViewState.DASHBOARD)}
                >
                    <div className="w-8 h-8 bg-space-accent rounded-lg flex items-center justify-center">
                        <LayoutGrid className="w-5 h-5 text-space-900" />
                    </div>
                    <span className="hidden sm:block">Nexus</span>
                </div>

                <div className="flex items-center gap-4 md:gap-8">
                    {/* Vault Button */}
                    <button 
                        onClick={() => setView(ViewState.VAULT)}
                        className="flex items-center gap-2 hover:bg-white/5 px-3 py-1.5 rounded-lg transition-colors"
                    >
                        <User className="w-4 h-4 text-space-light" />
                        <span className="text-xs font-bold hidden sm:inline">Profile</span>
                    </button>

                    {/* Level Badge */}
                    <div className="flex items-center gap-2">
                        <div className="w-8 h-8 rounded-full bg-space-800 border border-space-600 flex items-center justify-center text-xs font-bold text-cyan-400 shadow-inner">
                            {player.level}
                        </div>
                        <div className="flex flex-col">
                            <span className="text-[10px] text-space-light uppercase tracking-wider font-bold">Lvl</span>
                            <div className="w-24 h-1.5 bg-space-800 rounded-full overflow-hidden border border-space-700/50">
                                <div className="h-full bg-gradient-to-r from-cyan-400 to-green-500 transition-all duration-500" style={{ width: `${(player.xp % 300) / 3}%` }}></div>
                            </div>
                        </div>
                    </div>

                    {/* XP Counter */}
                    <div className="flex items-center gap-2 bg-space-900/80 px-3 py-1.5 rounded-lg border border-space-700/50">
                        <Star className="w-4 h-4 text-yellow-400 fill-current" />
                        <span className="font-mono font-bold text-white">{player.xp}</span>
                    </div>
                </div>
            </div>
        </div>
      )}

      <main className="min-h-[calc(100vh-64px)] flex flex-col">
        {loading ? (
            <div className="flex-1 flex flex-col items-center justify-center">import React, { useState, useEffect } from 'react';
import { Trophy, RefreshCcw, Gamepad2, Timer, AlertCircle, Zap } from 'lucide-react';

// --- GAME 1: MEMORY MATRIX ---
const MemoryGame = ({ onComplete }: { onComplete: () => void }) => {
  const icons = ['🤖', '🚀', '⭐', '🧬', '📐', '🧠', '⚡', '🔋'];
  const [cards, setCards] = useState<{ id: number; content: string; flipped: boolean; matched: boolean }[]>([]);
  const [flippedIndices, setFlippedIndices] = useState<number[]>([]);
  const [matchedCount, setMatchedCount] = useState(0);

  useEffect(() => {
    // Select 6 icons (12 cards total) for a quick game
    const gameIcons = icons.slice(0, 6);
    const doubled = [...gameIcons, ...gameIcons];
    const shuffled = doubled.sort(() => Math.random() - 0.5).map((icon, i) => ({
      id: i,
      content: icon,
      flipped: false,
      matched: false
    }));
    setCards(shuffled);
  }, []);

  useEffect(() => {
    if (flippedIndices.length === 2) {
      const [first, second] = flippedIndices;
      if (cards[first].content === cards[second].content) {
        setCards(prev => prev.map((c, i) => 
          i === first || i === second ? { ...c, matched: true } : c
        ));
        setMatchedCount(p => p + 1);
        setFlippedIndices([]);
      } else {
        setTimeout(() => {
          setCards(prev => prev.map((c, i) => 
            i === first || i === second ? { ...c, flipped: false } : c
          ));
          setFlippedIndices([]);
        }, 800);
      }
    }
  }, [flippedIndices, cards]);

  const handleCardClick = (index: number) => {
    if (flippedIndices.length >= 2 || cards[index].flipped || cards[index].matched) return;
    
    setCards(prev => prev.map((c, i) => i === index ? { ...c, flipped: true } : c));
    setFlippedIndices(prev => [...prev, index]);
  };

  if (matchedCount === 6) {
    return (
      <div className="text-center p-8 animate-in zoom-in">
        <Trophy className="w-20 h-20 text-yellow-400 mx-auto mb-4 animate-bounce" />
        <h2 className="text-3xl font-bold mb-2 text-white">Victory!</h2>
        <p className="mb-6 text-space-light">Memory modules synchronized.</p>
        <button onClick={onComplete} className="bg-space-accent hover:bg-cyan-300 text-space-900 px-8 py-3 rounded-xl font-bold transition-all transform hover:scale-105">
          Return to Hub
        </button>
      </div>
    );
  }

  return (
    <div className="w-full max-w-md mx-auto">
      <div className="grid grid-cols-4 gap-3">
        {cards.map((card, idx) => (
          <button
            key={card.id}
            onClick={() => handleCardClick(idx)}
            className={`aspect-square rounded-xl text-3xl flex items-center justify-center transition-all duration-300 transform ${
              card.flipped || card.matched 
                ? 'bg-gradient-to-br from-space-accent to-blue-500 rotate-y-180 scale-100 shadow-[0_0_15px_rgba(34,211,238,0.4)]' 
                : 'bg-space-800 border border-space-700 hover:bg-space-700 hover:scale-95'
            }`}
          >
            {(card.flipped || card.matched) ? card.content : ''}
          </button>
        ))}
      </div>
      <p className="text-center text-sm text-space-light mt-6">Find all matching pairs to win.</p>
    </div>
  );
};

// --- GAME 2: CYBER REFLEX ---
const ReflexGame = ({ onComplete }: { onComplete: () => void }) => {
    const [targetPos, setTargetPos] = useState({ top: '50%', left: '50%' });
    const [score, setScore] = useState(0);
    const [timeLeft, setTimeLeft] = useState(15);
    const [gameOver, setGameOver] = useState(false);
    const [gameStarted, setGameStarted] = useState(false);

    useEffect(() => {
        if (gameStarted && timeLeft > 0 && !gameOver) {
            const timer = setTimeout(() => setTimeLeft(t => t - 1), 1000);
            return () => clearTimeout(timer);
        } else if (timeLeft === 0) {
            setGameOver(true);
        }
    }, [timeLeft, gameOver, gameStarted]);

    const moveTarget = () => {
        setScore(s => s + 10);
        setTargetPos({
            top: `${Math.random() * 80 + 10}%`,
            left: `${Math.random() * 80 + 10}%`
        });
    };

    const startGame = () => {
        setGameStarted(true);
        moveTarget();
    }

    if (!gameStarted) {
        return (
             <div className="text-center p-10">
                <Timer className="w-16 h-16 text-space-accent mx-auto mb-4" />
                <h3 className="text-2xl font-bold text-white mb-2">Cyber Reflex</h3>
                <p className="text-space-light mb-6">Click the targets as fast as you can in 15 seconds!</p>
                <button onClick={startGame} className="bg-space-accent text-space-900 px-8 py-3 rounded-xl font-bold">Start Game</button>
            </div>
        )
    }

    if (gameOver) {
        return (
            <div className="text-center p-10 animate-in zoom-in">
                <Trophy className="w-20 h-20 text-yellow-400 mx-auto mb-4" />
                <h2 className="text-3xl font-bold text-white mb-2">Time's Up!</h2>
                <div className="text-5xl font-black text-transparent bg-clip-text bg-gradient-to-r from-cyan-400 to-green-400 mb-6">
                    {score} PTS
                </div>
                <button onClick={onComplete} className="bg-space-accent hover:bg-cyan-300 text-space-900 px-8 py-3 rounded-xl font-bold">
                    Collect Rewards
                </button>
            </div>
        );
    }

    return (
        <div className="relative w-full max-w-lg h-80 bg-space-900/50 rounded-2xl overflow-hidden border border-space-600 cursor-crosshair mx-auto shadow-inner">
            <div className="absolute top-4 left-4 right-4 flex justify-between text-lg font-mono font-bold text-white z-10 pointer-events-none">
                <span className="text-red-400">Time: {timeLeft}s</span>
                <span className="text-cyan-400">Score: {score}</span>
            </div>
            <button
                onMouseDown={moveTarget}
                style={{ top: targetPos.top, left: targetPos.left }}
                className="absolute w-14 h-14 bg-gradient-to-r from-red-500 to-orange-500 rounded-full shadow-[0_0_20px_rgba(239,68,68,0.6)] transform -translate-x-1/2 -translate-y-1/2 active:scale-90 transition-all duration-75 border-2 border-white"
            >
                <div className="absolute inset-0 bg-white opacity-20 rounded-full animate-ping"></div>
            </button>
        </div>
    );
}

interface MiniGamesProps {
    onExit: () => void;
}

const MiniGames: React.FC<MiniGamesProps> = ({ onExit }) => {
    const [selectedGame, setSelectedGame] = useState<'memory' | 'reflex' | null>(null);

    if (selectedGame === 'memory') {
        return <MemoryGame onComplete={onExit} />;
    }
    
    if (selectedGame === 'reflex') {
        return <ReflexGame onComplete={onExit} />;
    }

    return (
        <div className="max-w-2xl mx-auto text-center animate-in fade-in slide-in-from-bottom-4">
            <Gamepad2 className="w-16 h-16 text-space-accent mx-auto mb-4" />
            <h2 className="text-3xl md:text-4xl font-bold text-white mb-3">The Arcade</h2>
            <p className="text-space-light mb-8 max-w-md mx-auto">
                Excellent work on your worksheet! You've unlocked the game zone. Choose your reward.
            </p>

            <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                <button 
                    onClick={() => setSelectedGame('memory')}
                    className="group relative p-6 bg-space-800/60 rounded-2xl border border-space-700 hover:border-space-accent hover:bg-space-800 transition-all text-left overflow-hidden"
                >
                    <div className="absolute top-0 right-0 p-3 opacity-10 group-hover:opacity-20 transition-opacity">
                        <Timer size={100} />
                    </div>
                    <h3 className="text-xl font-bold text-white mb-2 group-hover:text-cyan-400">Memory Matrix</h3>
                    <p className="text-sm text-space-light">Match symbols to synchronize the system.</p>
                </button>

                <button 
                    onClick={() => setSelectedGame('reflex')}
                    className="group relative p-6 bg-space-800/60 rounded-2xl border border-space-700 hover:border-space-accent hover:bg-space-800 transition-all text-left overflow-hidden"
                >
                    <div className="absolute top-0 right-0 p-3 opacity-10 group-hover:opacity-20 transition-opacity">
                        <Zap size={100} /> {/* Assuming Zap is imported or exists */}
                    </div>
                    <h3 className="text-xl font-bold text-white mb-2 group-hover:text-red-400">Cyber Reflex</h3>
                    <p className="text-sm text-space-light">Test your reaction speed against moving targets.</p>
                </button>
            </div>
            
             <button onClick={onExit} className="mt-12 text-space-light hover:text-white underline">
                Skip Games & Return to Dashboard
            </button>
        </div>
    );
};

export default MiniGames;import React from 'react';
import { PlayerState, Avatar, Artifact } from '../types';
import { User, Shield, Zap, Box, Star, Lock, CircuitBoard } from 'lucide-react';

interface ProfileVaultProps {
  player: PlayerState;
  avatars: Avatar[];
  onSelectAvatar: (id: string) => void;
  onClose: () => void;
}

const ProfileVault: React.FC<ProfileVaultProps> = ({ player, avatars, onSelectAvatar, onClose }) => {
  return (
    <div className="max-w-4xl mx-auto p-6 animate-in fade-in slide-in-from-bottom-4">
      <div className="flex justify-between items-center mb-8">
        <h2 className="text-3xl font-bold bg-clip-text text-transparent bg-gradient-to-r from-space-light to-white">
          Access Vault
        </h2>
        <button onClick={onClose} className="text-space-light hover:text-white underline">
          Close Vault
        </button>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
        
        {/* Avatars Section */}
        <div className="bg-space-800/40 p-6 rounded-2xl border border-space-700 backdrop-blur-sm">
          <div className="flex items-center gap-2 mb-4">
            <User className="text-space-accent" />
            <h3 className="text-xl font-bold text-white">Identity Matrices</h3>
          </div>
          <p className="text-sm text-space-light mb-6">Unlock new digital identities by leveling up.</p>
          
          <div className="grid grid-cols-3 sm:grid-cols-4 gap-4">
            {avatars.map(avatar => {
              const isUnlocked = player.unlockedAvatars.includes(avatar.id);
              const isSelected = player.currentAvatar === avatar.id;
              
              return (
                <button
                  key={avatar.id}
                  disabled={!isUnlocked}
                  onClick={() => onSelectAvatar(avatar.id)}
                  className={`relative aspect-square rounded-xl border-2 flex flex-col items-center justify-center transition-all ${
                    isSelected 
                      ? 'border-space-accent bg-space-accent/20 shadow-[0_0_15px_rgba(34,211,238,0.5)]' 
                      : isUnlocked 
                        ? 'border-space-700 bg-space-900/60 hover:bg-space-800 hover:border-space-600' 
                        : 'border-space-800 bg-space-900/40 opacity-50 cursor-not-allowed'
                  }`}
                >
                  {!isUnlocked && (
                    <div className="absolute top-1 right-1">
                      <Lock size={12} className="text-space-light" />
                    </div>
                  )}
                  <div className={`${avatar.color} mb-1`}>
                    {avatar.icon === 'user' && <User />}
                    {avatar.icon === 'shield' && <Shield />}
                    {avatar.icon === 'zap' && <Zap />}
                    {avatar.icon === 'box' && <Box />}
                    {avatar.icon === 'star' && <Star />}
                  </div>
                  <span className="text-[10px] text-space-light font-mono truncate w-full px-2 text-center">
                    {isUnlocked ? avatar.name : `Lvl ${avatar.requiredLevel}`}
                  </span>
                </button>
              );
            })}
          </div>
        </div>

        {/* Artifacts Section */}
        <div className="bg-space-800/40 p-6 rounded-2xl border border-space-700 backdrop-blur-sm">
           <div className="flex items-center gap-2 mb-4">
            <CircuitBoard className="text-space-accent" />
            <h3 className="text-xl font-bold text-white">Relic Archive</h3>
          </div>
          <p className="text-sm text-space-light mb-6">Rare data fragments recovered from perfect runs.</p>

          {player.inventory.length === 0 ? (
            <div className="flex flex-col items-center justify-center h-48 text-space-light border border-dashed border-space-700 rounded-xl bg-space-900/20">
              <Box className="w-8 h-8 mb-2 opacity-50" />
              <p>No fragments recovered yet.</p>
              <p className="text-xs mt-1">Score 100% on quizzes to find them.</p>
            </div>
          ) : (
            <div className="space-y-3 max-h-[300px] overflow-y-auto pr-2 custom-scrollbar">
              {player.inventory.map((item, idx) => (
                <div key={idx} className="bg-space-900/60 border border-space-700 p-3 rounded-lg flex items-start gap-3 hover:border-space-600 transition-colors">
                  <div className={`p-2 rounded bg-space-800 ${item.rarity === 'Legendary' ? 'text-yellow-400' : 'text-cyan-400'}`}>
                    <Box size={20} />
                  </div>
                  <div>
                    <div className="flex items-center gap-2">
                      <h4 className="font-bold text-sm text-white">{item.name}</h4>
                      <span className={`text-[10px] px-1.5 py-0.5 rounded border ${
                        item.rarity === 'Legendary' 
                          ? 'border-yellow-500/50 text-yellow-400 bg-yellow-500/10' 
                          : 'border-cyan-500/50 text-cyan-400 bg-cyan-500/10'
                      }`}>
                        {item.rarity}
                      </span>
                    </div>
                    <p className="text-xs text-space-light mt-1">{item.description}</p>
                  </div>
                </div>
              ))}
            </div>
          )}
        </div>

      </div>
    </div>
  );
};

export default ProfileVault;import React, { useState, useEffect } from 'react';
import { QuizData, Question } from '../types';
import { CheckCircle, XCircle, ArrowRight, BrainCircuit, AlertTriangle, ShieldAlert, Star, Timer, BookOpenCheck } from 'lucide-react';

interface QuizArenaProps {
  quizData: QuizData;
  timeLimit?: number; // Optional time limit in seconds
  onComplete: (score: number, totalQuestions: number) => void;
  onExit: () => void;
}

const QuizArena: React.FC<QuizArenaProps> = ({ quizData, timeLimit, onComplete, onExit }) => {
  const [currentQIndex, setCurrentQIndex] = useState(0);
  const [selectedOption, setSelectedOption] = useState<number | null>(null);
  const [isAnswered, setIsAnswered] = useState(false);
  const [score, setScore] = useState(0);
  const [showResults, setShowResults] = useState(false);
  const [timeLeft, setTimeLeft] = useState<number | null>(timeLimit || null);

  const currentQuestion: Question = quizData.questions[currentQIndex];

  // Timer Logic
  useEffect(() => {
    if (timeLeft === null || showResults) return;

    if (timeLeft <= 0) {
      finishQuiz();
      return;
    }

    const timer = setInterval(() => {
      setTimeLeft(prev => (prev !== null ? prev - 1 : null));
    }, 1000);

    return () => clearInterval(timer);
  }, [timeLeft, showResults]);

  const handleOptionClick = (index: number) => {
    if (isAnswered) return;
    setSelectedOption(index);
    setIsAnswered(true);

    if (index === currentQuestion.correctIndex) {
      setScore(s => s + 1);
    }
  };

  const handleNext = () => {
    if (currentQIndex < quizData.questions.length - 1) {
      setCurrentQIndex(prev => prev + 1);
      setSelectedOption(null);
      setIsAnswered(false);
    } else {
      setShowResults(true);
    }
  };

  const finishQuiz = () => {
    setShowResults(true); 
  };

  const submitResults = () => {
    onComplete(score, quizData.questions.length);
  }

  const formatTime = (seconds: number) => {
    const mins = Math.floor(seconds / 60);
    const secs = seconds % 60;
    return `${mins}:${secs.toString().padStart(2, '0')}`;
  };

  if (showResults) {
    const passed = (score / quizData.questions.length) >= 0.5;
    return (
      <div className="flex flex-col items-center justify-center h-full p-8 animate-in fade-in zoom-in duration-500">
        <div className="glass-panel p-8 rounded-2xl border border-space-700 shadow-2xl max-w-md w-full text-center bg-space-800/80">
          <div className={`mx-auto mb-4 w-20 h-20 rounded-full flex items-center justify-center ${passed ? 'bg-green-500/20 text-green-400' : 'bg-red-500/20 text-red-400'}`}>
            {passed ? <Star className="w-10 h-10 fill-current" /> : <AlertTriangle className="w-10 h-10" />}
          </div>
          
          <h2 className="text-3xl font-bold mb-2 text-white">
            {passed ? "Worksheet Complete!" : "Keep Practicing"}
          </h2>
          <div className="text-6xl font-black text-transparent bg-clip-text bg-gradient-to-r from-cyan-300 to-green-300 mb-6">
            {Math.round((score / quizData.questions.length) * 100)}%
          </div>
          <p className="text-space-light mb-8">
            You answered {score} out of {quizData.questions.length} correctly.
          </p>
          <button
            onClick={submitResults}
            className={`w-full font-bold py-3 px-6 rounded-lg transition-all transform hover:scale-105 shadow-lg ${
                passed 
                ? 'bg-space-accent hover:bg-cyan-300 text-space-900 shadow-cyan-900/50'
                : 'bg-space-700 hover:bg-space-600 text-white'
            }`}
          >
            {passed ? "Claim Reward & Unlock Games" : "Return to Dashboard"}
          </button>
        </div>
      </div>
    );
  }

  return (
    <div className="max-w-2xl mx-auto w-full p-4">
      {/* Header */}
      <div className="flex justify-between items-center mb-6">
        <div className="flex flex-col">
          <div className="text-sm font-medium text-space-light uppercase tracking-wider opacity-80">
            {quizData.topic}
          </div>
          <div className={`flex items-center gap-1 text-xs font-bold border px-2 py-0.5 rounded-full w-fit mt-1 text-space-light border-space-600`}>
            <BookOpenCheck size={12} />
            <span>Grade {quizData.grade}</span>
          </div>
        </div>
        
        {/* Timer & Progress */}
        <div className="flex items-center gap-4">
          {timeLeft !== null && (
            <div className={`flex items-center gap-2 font-mono text-xl font-bold ${timeLeft < 10 ? 'text-red-400 animate-pulse' : 'text-cyan-300'}`}>
              <Timer className="w-5 h-5" />
              {formatTime(timeLeft)}
            </div>
          )}
          <div className="flex items-center gap-2">
            <span className="text-xs bg-space-800 px-2 py-1 rounded text-space-light border border-space-700">
              Q{currentQIndex + 1}/{quizData.questions.length}
            </span>
            <button onClick={onExit} className="text-space-light hover:text-white text-sm">Esc</button>
          </div>
        </div>
      </div>

      {/* Question Card */}
      <div className="glass-panel rounded-2xl border border-space-700 p-6 md:p-8 shadow-xl relative overflow-hidden bg-space-800/60">
        {/* Progress Bar Top */}
        <div className="absolute top-0 left-0 h-1 bg-space-900 w-full">
            <div 
                className="h-full bg-space-accent transition-all duration-300 shadow-[0_0_10px_#22d3ee]" 
                style={{ width: `${((currentQIndex) / quizData.questions.length) * 100}%`}}
            />
        </div>

        <h3 className="text-xl md:text-2xl font-bold text-white mb-6 leading-relaxed drop-shadow-sm">
          {currentQuestion.text}
        </h3>

        <div className="space-y-3">
          {currentQuestion.options.map((option, idx) => {
            let btnClass = "w-full text-left p-4 rounded-xl border transition-all duration-200 flex justify-between items-center group ";
            
            if (isAnswered) {
              if (idx === currentQuestion.correctIndex) {
                btnClass += "bg-green-900/60 border-green-400 text-green-100";
              } else if (idx === selectedOption) {
                btnClass += "bg-red-900/60 border-red-400 text-red-100";
              } else {
                btnClass += "bg-space-900/40 border-space-700 text-space-light opacity-50";
              }
            } else {
              btnClass += "bg-space-900/40 border-space-700 hover:border-space-accent hover:bg-space-800 text-space-light hover:text-white";
            }

            return (
              <button
                key={idx}
                onClick={() => handleOptionClick(idx)}
                disabled={isAnswered}
                className={btnClass}
              >
                <span>{option}</span>
                {isAnswered && idx === currentQuestion.correctIndex && <CheckCircle className="w-5 h-5 text-green-400" />}
                {isAnswered && idx === selectedOption && idx !== currentQuestion.correctIndex && <XCircle className="w-5 h-5 text-red-400" />}
              </button>
            );
          })}
        </div>

        {/* Feedback Area */}
        {isAnswered && (
          <div className="mt-6 p-4 bg-space-900/60 rounded-lg border border-space-600 animate-in fade-in slide-in-from-bottom-2">
            <div className="flex items-start gap-3">
              <div className="mt-1">
                {selectedOption === currentQuestion.correctIndex 
                  ? <CheckCircle className="w-5 h-5 text-green-400" />
                  : <AlertTriangle className="w-5 h-5 text-yellow-400" />
                }
              </div>
              <div>
                <h4 className="font-bold text-sm text-white mb-1">
                  {selectedOption === currentQuestion.correctIndex ? "Correct" : "Incorrect"}
                </h4>
                <p className="text-sm text-space-light leading-relaxed">
                  {currentQuestion.explanation}
                </p>
              </div>
            </div>
            <div className="mt-4 flex justify-end">
              <button
                onClick={handleNext}
                className="flex items-center gap-2 bg-space-accent hover:bg-cyan-300 text-space-900 px-6 py-2 rounded-lg font-bold transition-colors shadow-lg shadow-cyan-900/30"
              >
                {currentQIndex < quizData.questions.length - 1 ? "Next" : "Finish"}
                <ArrowRight className="w-4 h-4" />
              </button>
            </div>
          </div>
        )}
      </div>
    </div>
  );
};

export default QuizArena;<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Nexus: The Knowledge Grid</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script>
      tailwind.config = {
        theme: {
          extend: {
            colors: {
              space: {
                900: '#0c4a6e', // Sky 900 (Deep Blue)
                800: '#164e63', // Cyan 900 (Deep Teal)
                700: '#155e75', // Cyan 800 (Rich Teal)
                600: '#0891b2', // Cyan 600
                light: '#bae6fd', // Sky 200 (Pale Blue Text)
                accent: '#22d3ee', // Cyan 400 (Electric Blue)
                glow: '#2dd4bf', // Teal 400 (Bright Teal)
                success: '#34d399',
                warning: '#fbbf24',
                error: '#f87171'
              }
            },
            animation: {
              'pulse-slow': 'pulse 3s cubic-bezier(0.4, 0, 0.6, 1) infinite',
              'float': 'float 6s ease-in-out infinite',
              'glow': 'glow 2s ease-in-out infinite alternate',
            },
            keyframes: {
              float: {
                '0%, 100%': { transform: 'translateY(0)' },
                '50%': { transform: 'translateY(-10px)' },
              },
              glow: {
                '0%': { boxShadow: '0 0 5px #22d3ee' },
                '100%': { boxShadow: '0 0 20px #22d3ee, 0 0 10px #2dd4bf' }
              }
            }
          }
        }
      }
    </script>
    <style>
      body {
        background-color: #0c4a6e;
        background-image: 
          linear-gradient(150deg, #0c4a6e 0%, #115e59 100%);
        color: #e0f2fe;
        font-family: 'Inter', sans-serif;
        min-height: 100vh;
      }
      /* Custom Scrollbar */
      ::-webkit-scrollbar {
        width: 8px;
      }
      ::-webkit-scrollbar-track {
        background: #0c4a6e; 
      }
      ::-webkit-scrollbar-thumb {
        background: #164e63; 
        border-radius: 4px;
      }
      ::-webkit-scrollbar-thumb:hover {
        background: #22d3ee; 
      }
      .glass-panel {
        background: rgba(22, 78, 99, 0.6); /* Cyan 900 with opacity */
        backdrop-filter: blur(12px);
        -webkit-backdrop-filter: blur(12px);
        border: 1px solid rgba(34, 211, 238, 0.2); /* Cyan border */
      }
    </style>
  <script type="importmap">
{
  "imports": {
    "react/": "https://aistudiocdn.com/react@^19.2.0/",
    "react": "https://aistudiocdn.com/react@^19.2.0",
    "react-dom/": "https://aistudiocdn.com/react-dom@^19.2.0/",
    "@google/genai": "https://aistudiocdn.com/@google/genai@^1.30.0",
    "lucide-react": "https://aistudiocdn.com/lucide-react@^0.555.0"
  }
}
</script>
</head>
  <body>
    <div id="root"></div>
  </body>
</html>import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';

const rootElement = document.getElementById('root');
if (!rootElement) {
  throw new Error("Could not find root element to mount to");
}

const root = ReactDOM.createRoot(rootElement);
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
{
  "name": "Nexus: The Knowledge Grid",
  "description": "Travel through infinite dimensions of knowledge. From K-Pop stages to Ninja Villages, earn XP and unlock new fandoms in this gamified quiz RPG.",
  "requestFramePermissions": []
}export enum ViewState {
  GRADE_SELECT = 'GRADE_SELECT',
  DASHBOARD = 'DASHBOARD',
  QUIZ = 'QUIZ',
  ARCADE = 'ARCADE',
  VAULT = 'VAULT'
}

export interface PlayerState {
  xp: number;
  grade: number; // 1-12
  level: number;
  completedWorksheets: number;
  recentScores: number[]; 
  inventory: Artifact[];
  unlockedAvatars: string[];
  currentAvatar: string;
}

export interface Subject {
  id: string;
  name: string;
  topicKeyword: string; 
  description: string;
  icon: string; // lucide icon name
  color: string;
}

export interface Question {
  id: number;
  text: string;
  options: string[];
  correctIndex: number;
  explanation: string;
}

export interface QuizData {
  topic: string;
  grade: number;
  questions: Question[];
}

export interface GeminiQuizResponse {
  questions: {
    questionText: string;
    options: string[];
    correctAnswerIndex: number;
    explanation: string;
  }[];
}

export interface Artifact {
  id: string;
  name: string;
  description: string;
  rarity: 'Common' | 'Rare' | 'Legendary';
  dateAcquired: number;
}

export interface Avatar {
  id: string;
  name: string;
  icon: string;
  color: string;
  requiredLevel: number;
}
