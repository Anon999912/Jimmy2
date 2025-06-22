import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import {
    getAuth,
    onAuthStateChanged,
    signInAnonymously,
    signInWithCustomToken,
    signOut
} from 'firebase/auth';
import {
    getFirestore,
    collection,
    addDoc,
    query,
    onSnapshot,
    orderBy,
    serverTimestamp,
    doc,
    setDoc,
} from 'firebase/firestore';

// --- Firebase Configuration ---
// These variables are provided by the environment.
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-chat-app';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// --- Main App Component ---
export default function App() {
    const [user, setUser] = useState(null);
    const [auth, setAuth] = useState(null);
    const [db, setDb] = useState(null);
    const [isAuthReady, setIsAuthReady] = useState(false);

    // --- Initialize Firebase and Authentication ---
    useEffect(() => {
        try {
            const app = initializeApp(firebaseConfig);
            const authInstance = getAuth(app);
            const dbInstance = getFirestore(app);

            setAuth(authInstance);
            setDb(dbInstance);

            const unsubscribe = onAuthStateChanged(authInstance, (firebaseUser) => {
                setUser(firebaseUser);
                setIsAuthReady(true);
            });

            // Cleanup subscription on unmount
            return () => unsubscribe();
        } catch (error) {
            console.error("Error initializing Firebase:", error);
        }
    }, []);

    if (!isAuthReady) {
        return <LoadingScreen />;
    }

    return (
        <div className="flex flex-col h-screen bg-gray-900 text-white font-sans">
            {user ? <ChatRoom auth={auth} db={db} user={user} /> : <SignIn auth={auth} />}
        </div>
    );
}


// --- Loading Screen Component ---
function LoadingScreen() {
    return (
        <div className="flex items-center justify-center h-screen bg-gray-900">
            <div className="flex flex-col items-center">
                <svg className="animate-spin -ml-1 mr-3 h-10 w-10 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                    <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
                    <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                </svg>
                <p className="mt-4 text-lg">Connecting to the chat...</p>
            </div>
        </div>
    );
}

// --- Sign-In Component ---
function SignIn({ auth }) {
    const signIn = async () => {
        try {
            if (initialAuthToken) {
                await signInWithCustomToken(auth, initialAuthToken);
            } else {
                await signInAnonymously(auth);
            }
        } catch (error) {
            console.error("Error signing in:", error);
            // You might want to show an error message to the user here
        }
    };

    return (
        <div className="flex flex-col items-center justify-center h-full bg-gray-800 p-8 text-center">
            <h1 className="text-4xl font-bold mb-4">Welcome to the Real-time Messenger</h1>
            <p className="text-gray-300 mb-8 max-w-md">
                Chat with others in real-time. Click the button below to join the conversation anonymously. Your User ID will be generated for you.
            </p>
            <button
                onClick={signIn}
                className="px-8 py-4 bg-indigo-600 hover:bg-indigo-700 text-white font-bold rounded-full transition duration-300 ease-in-out transform hover:scale-105 shadow-lg"
            >
                Join Chat
            </button>
        </div>
    );
}

// --- Chat Room Component ---
function ChatRoom({ auth, db, user }) {
    const dummy = useRef();
    const [messages, setMessages] = useState([]);
    const [formValue, setFormValue] = useState('');
    
    // Construct the correct path for public data
    const messagesCollectionPath = `artifacts/${appId}/public/data/messages`;
    const messagesRef = collection(db, messagesCollectionPath);
    const q = query(messagesRef, orderBy('createdAt'));

    // --- Listen for new messages ---
    useEffect(() => {
        const unsubscribe = onSnapshot(q, (querySnapshot) => {
            const fetchedMessages = [];
            querySnapshot.forEach((doc) => {
                fetchedMessages.push({ id: doc.id, ...doc.data() });
            });
            setMessages(fetchedMessages);
        });

        return () => unsubscribe();
    }, [db]); // Re-run effect if db changes

    // --- Scroll to the bottom of the chat on new message ---
    useEffect(() => {
        dummy.current?.scrollIntoView({ behavior: 'smooth' });
    }, [messages]);
    
    // --- Send Message ---
    const sendMessage = async (e) => {
        e.preventDefault();
        if (formValue.trim() === '') return;

        const { uid } = auth.currentUser;
        
        try {
            await addDoc(messagesRef, {
                text: formValue,
                createdAt: serverTimestamp(),
                uid,
                displayName: `User-${uid.substring(0, 6)}` // Create a display name
            });
            setFormValue('');
            dummy.current?.scrollIntoView({ behavior: 'smooth' });
        } catch(error) {
            console.error("Error sending message: ", error);
        }
    };

    const handleSignOut = async () => {
        try {
            await signOut(auth);
        } catch (error) {
            console.error("Error signing out:", error);
        }
    };


    return (
        <>
            <header className="flex items-center justify-between p-4 bg-gray-800 border-b border-gray-700 shadow-md sticky top-0 z-10">
                <div className="flex flex-col">
                    <h1 className="text-xl font-bold text-white">Live Chat Room</h1>
                     <div className="flex items-center mt-1">
                        <span className="text-xs text-gray-400 mr-2">Your ID:</span>
                        <code className="text-xs bg-gray-700 text-indigo-300 px-2 py-1 rounded">{user.uid}</code>
                    </div>
                </div>
                <button onClick={handleSignOut} className="px-4 py-2 bg-red-600 hover:bg-red-700 text-white font-semibold rounded-lg transition duration-300 ease-in-out">
                    Sign Out
                </button>
            </header>

            <main className="flex-1 overflow-y-auto p-4 space-y-4">
                {messages && messages.map(msg => <ChatMessage key={msg.id} message={msg} currentUserUid={auth.currentUser.uid} />)}
                <span ref={dummy}></span>
            </main>

            <form onSubmit={sendMessage} className="flex items-center p-4 bg-gray-800 border-t border-gray-700 sticky bottom-0">
                <input
                    value={formValue}
                    onChange={(e) => setFormValue(e.target.value)}
                    placeholder="Say something nice"
                    className="flex-1 bg-gray-700 rounded-full py-3 px-6 text-white placeholder-gray-400 focus:outline-none focus:ring-2 focus:ring-indigo-500"
                />
                <button type="submit" disabled={!formValue.trim()} className="ml-4 p-3 bg-indigo-600 rounded-full text-white disabled:bg-gray-600 disabled:cursor-not-allowed hover:bg-indigo-700 transition duration-300 ease-in-out">
                    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" className="feather feather-send"><line x1="22" y1="2" x2="11" y2="13"></line><polygon points="22 2 15 22 11 13 2 9 22 2"></polygon></svg>
                </button>
            </form>
        </>
    );
}

// --- Chat Message Component ---
function ChatMessage({ message, currentUserUid }) {
    const { text, uid, displayName } = message;
    
    // Determine if the message was sent by the current user
    const messageClass = uid === currentUserUid ? 'sent' : 'received';

    return (
        <div className={`flex items-end ${messageClass === 'sent' ? 'justify-end' : 'justify-start'}`}>
            <div className={`flex flex-col space-y-1 max-w-xs lg:max-w-md ${messageClass === 'sent' ? 'items-end' : 'items-start'}`}>
                 <div
                    className={`px-4 py-3 rounded-2xl inline-block ${
                        messageClass === 'sent' 
                        ? 'bg-indigo-600 text-white rounded-br-none' 
                        : 'bg-gray-700 text-gray-200 rounded-bl-none'
                    }`}
                >
                    <p className="text-sm font-semibold mb-1 text-indigo-300">{displayName}</p>
                    <p className="break-words">{text}</p>
                </div>
            </div>
        </div>
    );
}

