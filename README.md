<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Real-Time Chat App</title>
    <script src="https://www.gstatic.com/firebasejs/9.15.0/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.15.0/firebase-firestore-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.15.0/firebase-auth-compat.js"></script>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Inter', sans-serif;
        }
        .message {
            @apply rounded-lg py-3 px-4 mb-2;
        }
        .sent {
            @apply bg-blue-500 text-white ml-auto max-w-[70%];
        }
        .received {
            @apply bg-gray-200 text-gray-800 mr-auto max-w-[70%];
        }
        .error-message {
            @apply text-red-500 text-sm mt-2;
        }
        .login-container {
            @apply bg-white shadow-lg rounded-lg p-6 max-w-md w-full;
        }
        .input-error {
            @apply border-red-500 focus:ring-red-500;
        }
    </style>
</head>
<body class="bg-gray-100 flex justify-center items-center min-h-screen">
    <div id="login-container" class="login-container">
        <h1 class="text-2xl font-semibold text-gray-800 mb-4 text-center">Login</h1>
        <div id="error-display" class="error-message hidden"></div>
        <div class="mb-4">
            <label for="username" class="block text-gray-700 text-sm font-bold mb-2">Username</label>
            <input type="text" id="login-username" placeholder="Enter your username" class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline">
        </div>
        <div class="mb-6">
            <label for="password" class="block text-gray-700 text-sm font-bold mb-2">Password</label>
            <input type="password" id="login-password" placeholder="Enter your password" class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline">
        </div>
        <button id="login-button" class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline w-full">Log In</button>
    </div>

    <div class="container bg-white shadow-lg rounded-lg p-6 max-w-2xl w-full hidden" id="chat-container">
        <h1 class="text-2xl font-semibold text-gray-800 mb-4 text-center">Real-Time Chat</h1>
        <div id="chat-messages" class="overflow-y-auto max-h-96 mb-4">
            </div>
        <div class="flex space-x-2">
            <input type="text" id="message-input" placeholder="Type your message..." class="flex-1 border rounded-md p-2 focus:outline-none focus:ring-2 focus:ring-blue-500">
            <button id="send-button" class="bg-blue-500 hover:bg-blue-600 text-white rounded-md p-2 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-opacity-50">Send</button>
        </div>
    </div>

    <script>
        // Firebase configuration (replace with your actual config)
        const firebaseConfig = {
            apiKey: "YOUR_API_KEY",
            authDomain: "YOUR_AUTH_DOMAIN",
            projectId: "YOUR_PROJECT_ID",
            storageBucket: "YOUR_STORAGE_BUCKET",
            messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
            appId: "YOUR_APP_ID",
            measurementId: "YOUR_MEASUREMENT_ID"
        };

        // Initialize Firebase
        firebase.initializeApp(firebaseConfig);
        const db = firebase.firestore();
        const auth = firebase.auth();
        const chatMessagesRef = db.collection('chatMessages');

        const loginContainer = document.getElementById('login-container');
        const chatContainer = document.getElementById('chat-container');
        const loginButton = document.getElementById('login-button');
        const loginUsernameInput = document.getElementById('login-username');
        const loginPasswordInput = document.getElementById('login-password');
        const chatMessages = document.getElementById('chat-messages');
        const messageInput = document.getElementById('message-input');
        const sendButton = document.getElementById('send-button');
        const errorMessageDisplay = document.getElementById('error-display');

        let currentUser = null;
        const correctUsername = "tayesh"; // Define the correct username
        const correctPassword = "password";  // Define the correct password

        function showError(message) {
            errorMessageDisplay.textContent = message;
            errorMessageDisplay.classList.remove('hidden');
            loginUsernameInput.classList.add('input-error');
            loginPasswordInput.classList.add('input-error');
        }

        function hideError() {
            errorMessageDisplay.classList.add('hidden');
            errorMessageDisplay.textContent = '';
            loginUsernameInput.classList.remove('input-error');
            loginPasswordInput.classList.remove('input-error');
        }

        // Function to add a message to the chat
        function addMessageToChat(message, isSent) {
            const messageDiv = document.createElement('div');
            messageDiv.classList.add('message');
            messageDiv.classList.add(isSent ? 'sent' : 'received');
            messageDiv.textContent = message;
            chatMessages.appendChild(messageDiv);
            // Scroll to the bottom to show the latest message
            chatMessages.scrollTop = chatMessages.scrollHeight;
        }

        // Function to send a message to Firestore
        function sendMessageToFirestore(message) {
            if (currentUser) {
                chatMessagesRef.add({
                    text: message,
                    timestamp: firebase.firestore.FieldValue.serverTimestamp(),
                    sender: currentUser.uid // Use the user's UID as the sender
                })
                .then(() => {
                    messageInput.value = ''; // Clear the input after sending
                })
                .catch((error) => {
                    console.error("Error sending message: ", error);
                });
            } else {
                console.error("No user is logged in.");
            }
        }

        // Event listener for the send button
        sendButton.addEventListener('click', () => {
            const messageText = messageInput.value.trim();
            if (messageText !== '') {
                sendMessageToFirestore(messageText);
            }
        });

        // Event listener for the enter key in the input field
        messageInput.addEventListener('keydown', (event) => {
            if (event.key === 'Enter') {
                event.preventDefault(); // Prevent the default Enter behavior (form submit)
                const messageText = messageInput.value.trim();
                if (messageText !== '') {
                    sendMessageToFirestore(messageText);
                }
            }
        });

        // Real-time listener for new messages from Firestore
        chatMessagesRef.orderBy('timestamp').onSnapshot((snapshot) => {
            snapshot.docChanges().forEach((change) => {
                if (change.type === 'added') {
                    const messageData = change.doc.data();
                    // Determine if the message was sent by the current user
                    const isSent = currentUser && messageData.sender === currentUser.uid;
                    addMessageToChat(messageData.text, isSent);
                }
            });
        });

        // Event listener for the login button
        loginButton.addEventListener('click', () => {
            const username = loginUsernameInput.value.trim();
            const password = loginPasswordInput.value.trim();

            if (username === correctUsername && password === correctPassword) {
                // Successful login
                hideError();
                loginContainer.classList.add('hidden');
                chatContainer.classList.remove('hidden');
                
                // Simulate a logged-in user.  In a real app, you would use Firebase Auth.
                currentUser = {
                    uid: 'tayesh-user-id',  //  Hardcoded user ID for this example
                    displayName: 'Tayesh' // And display name
                };

            } else {
                // Incorrect credentials
                showError('Invalid username or password.');
                loginUsernameInput.value = ''; // Clear the input fields
                loginPasswordInput.value = '';
            }
        });
    </script>
</body>
</html>

