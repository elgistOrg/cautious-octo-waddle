import React, { useState, useEffect, useRef, useCallback } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged, setPersistence, browserLocalPersistence } from 'firebase/auth';
import { getFirestore, doc, addDoc, setDoc, updateDoc, deleteDoc, onSnapshot, collection, query, Timestamp, serverTimestamp, setLogLevel } from 'firebase/firestore';

// --- Helper: Get App ID ---
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-kanban-app';

// --- Helper: Firebase Config ---
// Ensure __firebase_config is defined, otherwise provide a default or handle error
let firebaseConfig;
try {
    firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;
} catch (error) {
    console.error("Error parsing Firebase config:", error);
    firebaseConfig = null; // Fallback or error state
}

// --- Main App Component ---
function App() {
    // Firebase state
    const [auth, setAuth] = useState(null);
    const [db, setDb] = useState(null);
    const [userId, setUserId] = useState(null);
    const [isAuthReady, setIsAuthReady] = useState(false);

    // App state
    const [tasks, setTasks] = useState([]);
    const [isLoading, setIsLoading] = useState(true);
    const [showConfetti, setShowConfetti] = useState(false);
    const [showAddTaskModal, setShowAddTaskModal] = useState(false);
    const [showDeleteModal, setShowDeleteModal] = useState(false);
    const [taskToDelete, setTaskToDelete] = useState(null);
    const [error, setError] = useState(null);

    const appInitialized = useRef(false);

    // Initialize Firebase and Auth
    useEffect(() => {
        if (!firebaseConfig) {
            setError("Firebase configuration is missing or invalid. App cannot initialize.");
            setIsLoading(false);
            return;
        }
        if (appInitialized.current) return;
        appInitialized.current = true;

        try {
            const app = initializeApp(firebaseConfig);
            const firestoreDb = getFirestore(app);
            const firebaseAuth = getAuth(app);

            setDb(firestoreDb);
            setAuth(firebaseAuth);
            setLogLevel('debug'); // Optional: for Firestore logging

            // Set persistence to local
            setPersistence(firebaseAuth, browserLocalPersistence)
            .then(() => {
                 // Auth state listener
                const unsubscribe = onAuthStateChanged(firebaseAuth, async (user) => {
                    if (user) {
                        setUserId(user.uid);
                        setIsAuthReady(true);
                        console.log("User is signed in with UID:", user.uid);
                    } else {
                        console.log("No user signed in, attempting anonymous sign-in.");
                        try {
                            if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                                console.log("Attempting sign in with custom token.");
                                await signInWithCustomToken(firebaseAuth, __initial_auth_token);
                                // onAuthStateChanged will handle setting user
                            } else {
                                console.log("Attempting anonymous sign-in.");
                                await signInAnonymously(firebaseAuth);
                                // onAuthStateChanged will handle setting user
                            }
                        } catch (authError) {
                            console.error("Error during sign-in:", authError);
                            setError("Authentication failed. Please try again later.");
                            setIsAuthReady(true); // Still ready, but with an error
                        }
                    }
                });
                return unsubscribe;
            })
            .catch((persistenceError) => {
                console.error("Error setting auth persistence: ", persistenceError);
                setError("Could not set auth persistence. Data may not be saved across sessions.");
                setIsAuthReady(true); // Proceed but inform user
            });

        } catch (e) {
            console.error("Firebase initialization error:", e);
            setError("Failed to initialize the application. Please check the console for details.");
            setIsLoading(false);
        }
    }, []);


    // Fetch tasks from Firestore
    useEffect(() => {
        if (!isAuthReady || !db || !userId) {
            if(isAuthReady && !userId && !auth?.currentUser) { // if auth is ready but no user, means sign in failed
                 setIsLoading(false); // stop loading if auth failed
            }
            return;
        }

        setIsLoading(true);
        const tasksCollectionPath = `/artifacts/${appId}/users/${userId}/tasks`;
        const q = query(collection(db, tasksCollectionPath));

        const unsubscribe = onSnapshot(q, (querySnapshot) => {
            const tasksData = [];
            querySnapshot.forEach((doc) => {
                tasksData.push({ id: doc.id, ...doc.data() });
            });
            // Sort tasks by creation date (newest first) or by a specific order field if you add one
            tasksData.sort((a, b) => (a.createdAt?.toDate() || 0) - (b.createdAt?.toDate() || 0));
            setTasks(tasksData);
            setIsLoading(false);
            setError(null); // Clear previous errors on successful fetch
        }, (err) => {
            console.error("Error fetching tasks:", err);
            setError("Failed to load tasks. Please check your connection or try again later.");
            setIsLoading(false);
        });

        return () => unsubscribe();
    }, [isAuthReady, db, userId, auth]);


    // Columns for the Kanban board
    const columns = [
        { id: 'todo', title: 'To Do', color: 'bg-blue-100', textColor: 'text-blue-700' },
        { id: 'inprogress', title: 'In Progress', color: 'bg-yellow-100', textColor: 'text-yellow-700' },
        { id: 'done', title: 'Done', color: 'bg-green-100', textColor: 'text-green-700' },
    ];

    // --- Task Operations ---
    const handleAddTask = async (title, description) => {
        if (!db || !userId) {
            setError("Cannot add task: Database not ready or user not identified.");
            return;
        }
        try {
            const tasksCollectionPath = `/artifacts/${appId}/users/${userId}/tasks`;
            await addDoc(collection(db, tasksCollectionPath), {
                title,
                description,
                status: 'todo', // Default status
                createdAt: serverTimestamp(), // Use serverTimestamp for consistency
            });
            setShowAddTaskModal(false);
        } catch (err) {
            console.error("Error adding task:", err);
            setError("Failed to add task.");
        }
    };

    const handleUpdateTaskStatus = async (taskId, newStatus) => {
        if (!db || !userId) {
            setError("Cannot update task: Database not ready or user not identified.");
            return;
        }
        try {
            const taskDocRef = doc(db, `/artifacts/${appId}/users/${userId}/tasks`, taskId);
            await updateDoc(taskDocRef, { status: newStatus });
            if (newStatus === 'done') {
                setShowConfetti(true);
                setTimeout(() => setShowConfetti(false), 4000); // Confetti duration
            }
        } catch (err) {
            console.error("Error updating task status:", err);
            setError("Failed to update task status.");
        }
    };

    const handleDeleteTask = async () => {
        if (!db || !userId || !taskToDelete) {
            setError("Cannot delete task: Database not ready, user not identified, or task not specified.");
            return;
        }
        try {
            const taskDocRef = doc(db, `/artifacts/${appId}/users/${userId}/tasks`, taskToDelete.id);
            await deleteDoc(taskDocRef);
            setTaskToDelete(null);
            setShowDeleteModal(false);
        } catch (err) {
            console.error("Error deleting task:", err);
            setError("Failed to delete task.");
        }
    };

    // --- Drag and Drop Handlers ---
    const handleDragStart = (e, taskId) => {
        e.dataTransfer.setData('taskId', taskId);
    };

    const handleDragOver = (e) => {
        e.preventDefault(); // Necessary to allow dropping
    };

    const handleDrop = (e, newStatus) => {
        e.preventDefault();
        const taskId = e.dataTransfer.getData('taskId');
        const task = tasks.find(t => t.id === taskId);
        if (task && task.status !== newStatus) {
            handleUpdateTaskStatus(taskId, newStatus);
        }
    };
    
    // --- UI Rendering ---
    if (!firebaseConfig) {
        return (
            <div className="min-h-screen bg-red-50 flex flex-col items-center justify-center p-4">
                <div className="bg-white p-8 rounded-lg shadow-xl text-center">
                    <h1 className="text-2xl font-bold text-red-700 mb-4">Application Error</h1>
                    <p className="text-red-600">{error || "Firebase configuration is missing. The application cannot start."}</p>
                </div>
            </div>
        );
    }


    return (
        <div className="min-h-screen bg-gray-100 font-sans p-4 md:p-8">
            {showConfetti && <ConfettiCanvas />}
            
            <header className="mb-8 text-center">
                <h1 className="text-4xl font-bold text-indigo-600">Kanban Task Tracker</h1>
                {userId && <p className="text-sm text-gray-500 mt-2">User ID: {userId}</p>}
                 {error && <p className="text-sm text-red-500 bg-red-100 p-2 rounded mt-2 inline-block">{error}</p>}
            </header>

            {isLoading && !isAuthReady && (
                 <div className="flex justify-center items-center h-64">
                    <div className="animate-spin rounded-full h-16 w-16 border-t-4 border-b-4 border-indigo-500"></div>
                    <p className="ml-4 text-indigo-700">Initializing Application...</p>
                </div>
            )}

            {isAuthReady && !userId && !isLoading && (
                 <div className="text-center text-red-600 bg-red-100 p-4 rounded-lg shadow">
                    <p>Authentication failed. Could not sign in user. Please refresh or check console.</p>
                </div>
            )}


            {isAuthReady && userId && (
                <>
                    <div className="mb-6 flex justify-center">
                        <button
                            onClick={() => setShowAddTaskModal(true)}
                            className="bg-indigo-500 hover:bg-indigo-600 text-white font-semibold py-3 px-6 rounded-lg shadow-md transition duration-150 ease-in-out transform hover:scale-105 focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:ring-opacity-50"
                        >
                            Add New Task
                        </button>
                    </div>

                    {isLoading && tasks.length === 0 && (
                        <div className="flex justify-center items-center h-64">
                            <div className="animate-spin rounded-full h-12 w-12 border-t-2 border-b-2 border-indigo-500"></div>
                            <p className="ml-3 text-gray-600">Loading tasks...</p>
                        </div>
                    )}

                    {!isLoading && tasks.length === 0 && (
                        <div className="text-center text-gray-500 mt-10">
                            <svg xmlns="http://www.w3.org/2000/svg" className="h-16 w-16 mx-auto text-gray-400" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z" />
                            </svg>
                            <p className="mt-2 text-xl">No tasks yet. Add one to get started!</p>
                        </div>
                    )}

                    <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
                        {columns.map(column => (
                            <div
                                key={column.id}
                                className={`p-4 rounded-lg shadow-lg ${column.color} min-h-[300px]`}
                                onDragOver={handleDragOver}
                                onDrop={(e) => handleDrop(e, column.id)}
                            >
                                <h2 className={`text-2xl font-semibold mb-4 pb-2 border-b-2 ${column.textColor} border-opacity-50 ${column.textColor.replace('text-', 'border-')}`}>{column.title}</h2>
                                <div className="space-y-3">
                                    {tasks.filter(task => task.status === column.id).map(task => (
                                        <TaskCard
                                            key={task.id}
                                            task={task}
                                            onDragStart={handleDragStart}
                                            onDelete={() => { setTaskToDelete(task); setShowDeleteModal(true); }}
                                        />
                                    ))}
                                    {tasks.filter(task => task.status === column.id).length === 0 && !isLoading && (
                                        <p className={`italic ${column.textColor} opacity-70`}>No tasks in this stage.</p>
                                    )}
                                </div>
                            </div>
                        ))}
                    </div>
                </>
            )}

            {showAddTaskModal && (
                <AddTaskModal
                    onClose={() => setShowAddTaskModal(false)}
                    onAddTask={handleAddTask}
                />
            )}

            {showDeleteModal && taskToDelete && (
                <DeleteConfirmationModal
                    taskTitle={taskToDelete.title}
                    onClose={() => { setShowDeleteModal(false); setTaskToDelete(null);}}
                    onConfirm={handleDeleteTask}
                />
            )}
        </div>
    );
}

// --- TaskCard Component ---
function TaskCard({ task, onDragStart, onDelete }) {
    return (
        <div
            draggable
            onDragStart={(e) => onDragStart(e, task.id)}
            className="bg-white p-4 rounded-md shadow-md cursor-grab hover:shadow-lg transition-shadow duration-150"
        >
            <h3 className="font-semibold text-gray-800 break-words">{task.title}</h3>
            {task.description && <p className="text-sm text-gray-600 mt-1 break-words">{task.description}</p>}
            {task.createdAt && (
                <p className="text-xs text-gray-400 mt-2">
                    Created: {task.createdAt.toDate ? task.createdAt.toDate().toLocaleDateString() : new Date(task.createdAt.seconds * 1000).toLocaleDateString()}
                </p>
            )}
            <button
                onClick={onDelete}
                className="mt-3 text-xs text-red-500 hover:text-red-700 transition-colors"
                aria-label={`Delete task ${task.title}`}
            >
                <svg xmlns="http://www.w3.org/2000/svg" className="h-4 w-4 inline mr-1" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                    <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
                </svg>
                Delete
            </button>
        </div>
    );
}

// --- AddTaskModal Component ---
function AddTaskModal({ onClose, onAddTask }) {
    const [title, setTitle] = useState('');
    const [description, setDescription] = useState('');
    const titleInputRef = useRef(null);

    useEffect(() => {
        titleInputRef.current?.focus();
    }, []);

    const handleSubmit = (e) => {
        e.preventDefault();
        if (!title.trim()) {
            alert("Title is required."); // Simple validation, could be improved with a nicer message
            return;
        }
        onAddTask(title, description);
        setTitle('');
        setDescription('');
    };

    return (
        <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50">
            <div className="bg-white p-6 rounded-lg shadow-xl w-full max-w-md">
                <h2 className="text-2xl font-semibold mb-4 text-gray-800">Add New Task</h2>
                <form onSubmit={handleSubmit}>
                    <div className="mb-4">
                        <label htmlFor="taskTitle" className="block text-sm font-medium text-gray-700 mb-1">Title <span className="text-red-500">*</span></label>
                        <input
                            type="text"
                            id="taskTitle"
                            ref={titleInputRef}
                            value={title}
                            onChange={(e) => setTitle(e.target.value)}
                            className="w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500"
                            required
                        />
                    </div>
                    <div className="mb-6">
                        <label htmlFor="taskDescription" className="block text-sm font-medium text-gray-700 mb-1">Description</label>
                        <textarea
                            id="taskDescription"
                            value={description}
                            onChange={(e) => setDescription(e.target.value)}
                            rows="3"
                            className="w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500"
                        ></textarea>
                    </div>
                    <div className="flex justify-end space-x-3">
                        <button
                            type="button"
                            onClick={onClose}
                            className="px-4 py-2 text-sm font-medium text-gray-700 bg-gray-100 hover:bg-gray-200 rounded-md border border-gray-300 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500"
                        >
                            Cancel
                        </button>
                        <button
                            type="submit"
                            className="px-4 py-2 text-sm font-medium text-white bg-indigo-600 hover:bg-indigo-700 rounded-md shadow-sm focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500"
                        >
                            Add Task
                        </button>
                    </div>
                </form>
            </div>
        </div>
    );
}

// --- DeleteConfirmationModal Component ---
function DeleteConfirmationModal({ taskTitle, onClose, onConfirm }) {
     return (
        <div className="fixed inset-0 bg-black bg-opacity-60 flex items-center justify-center p-4 z-50">
            <div className="bg-white p-6 sm:p-8 rounded-lg shadow-xl w-full max-w-md transform transition-all">
                <div className="text-center">
                    <div className="mx-auto flex items-center justify-center h-12 w-12 rounded-full bg-red-100 mb-4">
                        <svg className="h-6 w-6 text-red-600" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor" aria-hidden="true">
                            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z" />
                        </svg>
                    </div>
                    <h3 className="text-lg leading-6 font-medium text-gray-900" id="modal-title">
                        Delete Task
                    </h3>
                    <div className="mt-2">
                        <p className="text-sm text-gray-500">
                            Are you sure you want to delete the task "<strong>{taskTitle}</strong>"? This action cannot be undone.
                        </p>
                    </div>
                </div>
                <div className="mt-6 flex flex-col sm:flex-row-reverse sm:gap-3">
                    <button
                        type="button"
                        onClick={onConfirm}
                        className="w-full inline-flex justify-center rounded-md border border-transparent shadow-sm px-4 py-2 bg-red-600 text-base font-medium text-white hover:bg-red-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-red-500 sm:ml-3 sm:w-auto sm:text-sm mb-2 sm:mb-0"
                    >
                        Delete
                    </button>
                    <button
                        type="button"
                        onClick={onClose}
                        className="w-full inline-flex justify-center rounded-md border border-gray-300 shadow-sm px-4 py-2 bg-white text-base font-medium text-gray-700 hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500 sm:w-auto sm:text-sm"
                    >
                        Cancel
                    </button>
                </div>
            </div>
        </div>
    );
}


// --- ConfettiCanvas Component ---
const ConfettiCanvas = () => {
    const canvasRef = useRef(null);
    const animationFrameId = useRef(null);
    const particles = useRef([]);

    const Particle = useCallback((canvas) => {
        const colors = ["#f9a8d4", "#f472b6", "#ec4899", "#db2777", "#be185d", "#a3e635", "#84cc16", "#65a30d", "#4d7c0f"];
        const types = ['circle', 'square', 'triangle'];
        
        this.x = Math.random() * canvas.width;
        this.y = Math.random() * canvas.height * 0.2 - 50; // Start above screen
        this.size = Math.random() * 8 + 4; // Size between 4 and 12
        this.color = colors[Math.floor(Math.random() * colors.length)];
        this.type = types[Math.floor(Math.random() * types.length)];
        
        this.vx = Math.random() * 4 - 2; // Horizontal velocity
        this.vy = Math.random() * 3 + 2; // Vertical velocity (downwards)
        this.opacity = 1;
        this.rotation = Math.random() * 360;
        this.rotationSpeed = Math.random() * 2 - 1;

        this.update = () => {
            this.x += this.vx;
            this.y += this.vy;
            this.opacity -= 0.01; // Fade out
            this.vy += 0.05; // Gravity
            this.rotation += this.rotationSpeed;
        };

        this.draw = (ctx) => {
            ctx.save();
            ctx.globalAlpha = this.opacity;
            ctx.fillStyle = this.color;
            ctx.translate(this.x + this.size / 2, this.y + this.size / 2);
            ctx.rotate(this.rotation * Math.PI / 180);
            ctx.translate(-(this.x + this.size / 2), -(this.y + this.size / 2));

            if (this.type === 'circle') {
                ctx.beginPath();
                ctx.arc(this.x + this.size / 2, this.y + this.size / 2, this.size / 2, 0, Math.PI * 2);
                ctx.fill();
            } else if (this.type === 'square') {
                ctx.fillRect(this.x, this.y, this.size, this.size);
            } else if (this.type === 'triangle') {
                ctx.beginPath();
                ctx.moveTo(this.x + this.size / 2, this.y);
                ctx.lineTo(this.x + this.size, this.y + this.size);
                ctx.lineTo(this.x, this.y + this.size);
                ctx.closePath();
                ctx.fill();
            }
            ctx.restore();
        };
    }, []);


    useEffect(() => {
        const canvas = canvasRef.current;
        if (!canvas) return;
        const ctx = canvas.getContext('2d');
        
        let Dpr = window.devicePixelRatio || 1;
        canvas.width = window.innerWidth * Dpr;
        canvas.height = window.innerHeight * Dpr;
        ctx.scale(Dpr,Dpr);
        canvas.style.width = `${window.innerWidth}px`;
        canvas.style.height = `${window.innerHeight}px`;


        particles.current = [];
        for (let i = 0; i < 150; i++) { // Number of confetti particles
            particles.current.push(new Particle(canvas));
        }

        const animate = () => {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
            particles.current = particles.current.filter(p => p.opacity > 0 && p.y < canvas.height / Dpr);
            
            particles.current.forEach(particle => {
                particle.update();
                particle.draw(ctx);
            });

            if (particles.current.length > 0) {
                animationFrameId.current = requestAnimationFrame(animate);
            }
        };

        animate();

        const handleResize = () => {
            if (!canvasRef.current) return;
            Dpr = window.devicePixelRatio || 1;
            canvasRef.current.width = window.innerWidth * Dpr;
            canvasRef.current.height = window.innerHeight * Dpr;
            ctx.scale(Dpr,Dpr);
            canvasRef.current.style.width = `${window.innerWidth}px`;
            canvasRef.current.style.height = `${window.innerHeight}px`;
        };

        window.addEventListener('resize', handleResize);

        return () => {
            cancelAnimationFrame(animationFrameId.current);
            window.removeEventListener('resize', handleResize);
        };
    }, [Particle]);

    return (
        <canvas
            ref={canvasRef}
            style={{
                position: 'fixed',
                top: 0,
                left: 0,
                width: '100vw',
                height: '100vh',
                pointerEvents: 'none', // Make canvas non-interactive
                zIndex: 1000, // Ensure it's on top
            }}
        />
    );
};

export default App;

