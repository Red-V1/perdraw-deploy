<script lang="ts">
    import { fade, slide } from "svelte/transition";
    import { onMount } from "svelte";
    import Peer from "peerjs";

    let canvas: HTMLCanvasElement;
    let ctx: CanvasRenderingContext2D;
    let isDrawing = false;
    let lastX = 0;
    let lastY = 0;

    let peer: Peer;
    let connections: { [key: string]: Peer.DataConnection } = {};
    let peerId = "";
    let remotePeerId = "";
    let isHost = false;

    let brushSize = 2;
    let brushColor = "#000000";
    let isRainbowMode = false;
    let hue = 0;

    let animations: {
        id: number;
        duration: number;
        delay: number;
        x: number;
        y: number;
    }[] = [];
    let headerText = "PerDraw";
    let partyMode = false;
    let headerAnimation: number;
    let devMode = false;

    let shareableLink = "";

    // Chat-related variables
    let chatMessages: { sender: string; message: string }[] = [];
    let currentMessage = "";
    let isChatVisible = false;

    let displayName = "Anonymous";
    let isControlsVisible = false;
    let isMobile = false;

    function resizeCanvas() {
        if (canvas) {
            canvas.width = window.innerWidth;
            canvas.height = isMobile
                ? window.innerHeight - 60
                : window.innerHeight;
            if (ctx) {
                ctx.fillStyle = "white";
                ctx.fillRect(0, 0, canvas.width, canvas.height);
            }
        }
    }

    function setupConnection(conn: Peer.DataConnection) {
        conn.on("data", (data: any) => {
            if (data.type === "draw") {
                drawLine(
                    data.x1,
                    data.y1,
                    data.x2,
                    data.y2,
                    data.color,
                    data.size,
                );
                if (isHost) {
                    // Broadcast to all other connections
                    Object.values(connections).forEach((c) => {
                        if (c.peer !== conn.peer) {
                            c.send(data);
                        }
                    });
                }
            } else if (data.type === "chat") {
                chatMessages = [
                    ...chatMessages,
                    { sender: data.sender, message: data.message },
                ];
                if (isHost) {
                    // Broadcast to all other connections
                    Object.values(connections).forEach((c) => {
                        if (c.peer !== conn.peer) {
                            c.send(data);
                        }
                    });
                }
            }
        });
    }

    function connect(targetPeerId: string = remotePeerId) {
        if (peer && targetPeerId) {
            const conn = peer.connect(targetPeerId);
            conn.on("open", () => {
                setupConnection(conn);
                connections[conn.peer] = conn;
                isHost = false;
            });
        }
    }

    function startDrawing(e: MouseEvent | TouchEvent) {
        isDrawing = true;
        const pos = getEventPosition(e);
        [lastX, lastY] = [pos.x, pos.y];
    }

    function draw(e: MouseEvent | TouchEvent) {
        if (!isDrawing) return;
        const pos = getEventPosition(e);
        let currentColor = isRainbowMode
            ? `hsl(${hue}, 100%, 50%)`
            : brushColor;
        drawLine(lastX, lastY, pos.x, pos.y, currentColor, brushSize);
        sendData({
            type: "draw",
            x1: lastX,
            y1: lastY,
            x2: pos.x,
            y2: pos.y,
            color: currentColor,
            size: brushSize,
        });
        [lastX, lastY] = [pos.x, pos.y];

        if (isRainbowMode) {
            hue = (hue + 1) % 360;
        }
    }

    function stopDrawing() {
        isDrawing = false;
    }

    function getEventPosition(e: MouseEvent | TouchEvent) {
        let x, y;
        if (e instanceof MouseEvent) {
            x = e.offsetX;
            y = e.offsetY;
        } else {
            const rect = canvas.getBoundingClientRect();
            x = e.touches[0].clientX - rect.left;
            y = e.touches[0].clientY - rect.top;
        }
        return { x, y };
    }

    function drawLine(
        x1: number,
        y1: number,
        x2: number,
        y2: number,
        color: string,
        size: number,
    ) {
        if (ctx) {
            ctx.beginPath();
            ctx.moveTo(x1, y1);
            ctx.lineTo(x2, y2);
            ctx.strokeStyle = color;
            ctx.lineWidth = size;
            ctx.lineCap = "round";
            ctx.stroke();
        }
    }

    function sendData(data: any) {
        if (isHost) {
            Object.values(connections).forEach((conn) => conn.send(data));
        } else {
            Object.values(connections)[0]?.send(data);
        }
    }

    async function copyPeerId() {
        try {
            await navigator.clipboard.writeText(peerId);
            alert("Peer ID copied to clipboard!");
        } catch (err) {
            console.error("Failed to copy: ", err);
            alert("Failed to copy. Your Peer ID is: " + peerId);
        }
    }

    function togglePartyMode() {
        partyMode = !partyMode;
        if (partyMode) {
            startPartyMode();
            alert("YOOOOOOOOOO");
        } else {
            stopPartyMode();
            alert("YOUR FUCKING BORING");
        }
    }

    function startPartyMode() {
        headerAnimation = setInterval(() => {
            headerText = headerText
                .split("")
                .sort(() => Math.random() - 0.5)
                .join("");
        }, 1000);
    }

    function stopPartyMode() {
        clearInterval(headerAnimation);
        headerText = "PerDraw";
    }

    function sendDevMessage(message: string) {
        if (devMode) {
            alert("PerRed Says: " + message);
        }
    }

    function toggleRainbowMode() {
        isRainbowMode = !isRainbowMode;
    }

    function generateShareableLink() {
        const url = new URL(window.location.href);
        url.searchParams.set("connect", peerId);
        shareableLink = url.toString();
    }

    function sendChatMessage() {
        if (currentMessage.trim()) {
            const messageData = {
                type: "chat",
                sender: displayName,
                message: currentMessage.trim(),
            };
            sendData(messageData);
            chatMessages = [
                ...chatMessages,
                { sender: "You", message: currentMessage.trim() },
            ];
            currentMessage = "";
        }
    }

    function toggleChat() {
        isChatVisible = !isChatVisible;
    }

    function toggleControls() {
        isControlsVisible = !isControlsVisible;
    }

    onMount(() => {
        ctx = canvas.getContext("2d")!;
        resizeCanvas();
        window.addEventListener("resize", resizeCanvas);

        peer = new Peer();

        peer.on("open", (id) => {
            peerId = id;
            generateShareableLink();
            isHost = true;

            // Check if there's a 'connect' parameter in the URL
            const urlParams = new URLSearchParams(window.location.search);
            const connectTo = urlParams.get("connect");
            if (connectTo) {
                remotePeerId = connectTo;
                connect(connectTo);
            }
        });

        peer.on("connection", (conn) => {
            setupConnection(conn);
            connections[conn.peer] = conn;
        });

        for (let i = 0; i < 100; i++) {
            animations.push({
                id: i,
                duration: Math.random() * 5 + 1,
                delay: Math.random() * 2,
                x: Math.random() * 100,
                y: Math.random() * 100,
            });
        }

        // Dev mode console commands
        (window as any).activateDevMode = () => {
            devMode = true;
            console.log("Dev mode activated");
        };

        (window as any).deactivateDevMode = () => {
            devMode = false;
            console.log("Dev mode deactivated");
        };

        (window as any).sendDevMessage = (message: string) => {
            sendDevMessage(message);
        };

        // Check if the device is mobile
        isMobile =
            /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(
                navigator.userAgent,
            );

        // Adjust canvas size for mobile devices
        if (isMobile) {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight - 60; // Leave space for the toggle button
        }
    });
</script>

{#if partyMode}
    <header class="jarring-header">{headerText}</header>
{/if}

<main class:mobile={isMobile}>
    <canvas
        bind:this={canvas}
        on:mousedown={startDrawing}
        on:mousemove={draw}
        on:mouseup={stopDrawing}
        on:mouseout={stopDrawing}
        on:touchstart={startDrawing}
        on:touchmove={draw}
        on:touchend={stopDrawing}
    ></canvas>

    <button class="toggle-controls" on:click={toggleControls}>
        {isControlsVisible ? "✕" : "☰"}
    </button>

    {#if isControlsVisible}
        <div class="controls" transition:slide>
            <div class="control-section">
                <input
                    bind:value={displayName}
                    placeholder="Enter display name"
                />
            </div>
            <div class="control-section">
                <div class="connection-info">
                    Your ID: {peerId}
                    <button on:click={copyPeerId}>Copy ID</button>
                    <input
                        bind:value={remotePeerId}
                        placeholder="Enter peer ID to connect"
                    />
                    <button on:click={() => connect()}>Connect</button>
                    <button
                        on:click={() =>
                            navigator.clipboard.writeText(shareableLink)}
                    >
                        Copy Shareable Link
                    </button>
                </div>
            </div>
            <div class="control-section">
                <div class="drawing-controls">
                    <input
                        type="color"
                        bind:value={brushColor}
                        disabled={isRainbowMode}
                    />
                    <input
                        type="range"
                        bind:value={brushSize}
                        min="1"
                        max="50"
                    />
                    <span>Brush Size: {brushSize}</span>
                    <button on:click={toggleRainbowMode}>
                        {isRainbowMode ? "Disable" : "Enable"} Rainbow Mode
                    </button>
                </div>
            </div>
            <div class="control-section">
                <button on:click={togglePartyMode}>
                    {partyMode ? "Disable" : "Enable"} Party Mode
                </button>
            </div>
            <div class="control-section">
                <button on:click={toggleChat}>
                    {isChatVisible ? "Hide" : "Show"} Chat
                </button>
            </div>
        </div>
    {/if}

    {#if isChatVisible}
        <div class="chat-container" transition:slide>
            <div class="chat-messages">
                {#each chatMessages as message}
                    <div class="chat-message" transition:fade>
                        <strong>{message.sender}:</strong>
                        {message.message}
                    </div>
                {/each}
            </div>
            <div class="chat-input">
                <input
                    type="text"
                    bind:value={currentMessage}
                    on:keypress={(e) => e.key === "Enter" && sendChatMessage()}
                    placeholder="Type a message..."
                />
                <button on:click={sendChatMessage}>Send</button>
            </div>
        </div>
    {/if}
</main>

{#if partyMode}
    <footer class="jarring-footer"></footer>

    {#each animations as anim (anim.id)}
        <div
            class="floating-element"
            style="
        animation-duration: {anim.duration}s;
        animation-delay: {anim.delay}s;
        left: {anim.x}%;
        top: {anim.y}%;
      "
        ></div>
    {/each}
{/if}

<style>
    @keyframes float {
        0%,
        100% {
            transform: translate(0, 0) rotate(0deg);
        }
        25% {
            transform: translate(50px, -50px) rotate(90deg);
        }
        50% {
            transform: translate(-50px, 50px) rotate(180deg);
        }
        75% {
            transform: translate(50px, 50px) rotate(270deg);
        }
    }

    @keyframes colorShift {
        0% {
            background-color: red;
        }
        33% {
            background-color: green;
        }
        66% {
            background-color: blue;
        }
        100% {
            background-color: red;
        }
    }

    @keyframes shake {
        0%,
        100% {
            transform: translateX(0) translateY(0);
        }
        25% {
            transform: translateX(-5px) translateY(5px);
        }
        50% {
            transform: translateX(5px) translateY(-5px);
        }
        75% {
            transform: translateX(-5px) translateY(-5px);
        }
    }

    :global(body) {
        margin: 0;
        padding: 0;
        overflow: hidden;
        background-color: black;
        color: white;
        font-family: "Comic Sans MS", cursive;
    }

    header,
    footer {
        padding: 20px;
        text-align: center;
        font-size: 3em;
        position: fixed;
        width: 100%;
        z-index: 3;
    }

    header {
        top: 0;
    }

    footer {
        bottom: 0;
    }

    .jarring-header,
    .jarring-footer {
        animation:
            colorShift 5s infinite,
            shake 0.3s infinite;
        text-shadow:
            2px 2px 4px rgba(255, 255, 255, 0.5),
            -2px -2px 4px rgba(255, 255, 255, 0.5);
    }

    .jarring-header {
        transform-origin: center;
        animation:
            colorShift 5s infinite,
            shake 0.3s infinite,
            rotate 10s infinite;
    }

    @keyframes rotate {
        0% {
            transform: rotate(0deg);
        }
        100% {
            transform: rotate(360deg);
        }
    }

    .floating-element {
        position: fixed;
        width: 20px;
        height: 20px;
        background-color: rgba(255, 255, 255, 0.5);
        border-radius: 50%;
        animation: float 5s infinite;
        pointer-events: none;
    }

    @keyframes float {
        0%,
        100% {
            transform: translate(0, 0) rotate(0deg);
        }
        25% {
            transform: translate(50px, -50px) rotate(90deg);
        }
        50% {
            transform: translate(-50px, 50px) rotate(180deg);
        }
        75% {
            transform: translate(50px, 50px) rotate(270deg);
        }
    }

    main {
        position: relative;
        width: 100vw;
        height: 100vh;
        overflow: hidden;
    }

    main.mobile {
        height: calc(100vh - 60px);
    }

    .toggle-controls {
        position: fixed;
        top: 10px;
        left: 10px;
        z-index: 10;
        font-size: 24px;
        background-color: rgba(0, 0, 0, 0.7);
        color: white;
        border: none;
        border-radius: 50%;
        width: 40px;
        height: 40px;
        display: flex;
        align-items: center;
        justify-content: center;
        cursor: pointer;
    }

    .controls {
        position: fixed;
        top: 60px;
        left: 10px;
        z-index: 5;
        background-color: rgba(0, 0, 0, 0.7);
        padding: 10px;
        border-radius: 5px;
        max-width: 300px;
        max-height: calc(100vh - 80px);
        overflow-y: auto;
    }

    .control-section {
        margin-bottom: 10px;
    }

    .control-section:last-child {
        margin-bottom: 0;
    }

    .chat-container {
        position: fixed;
        right: 10px;
        bottom: 10px;
        width: 300px;
        height: 400px;
        background-color: rgba(0, 0, 0, 0.7);
        border-radius: 5px;
        display: flex;
        flex-direction: column;
        z-index: 5;
    }

    .chat-messages {
        flex-grow: 1;
        overflow-y: auto;
        padding: 10px;
    }

    .chat-message {
        margin-bottom: 5px;
        animation: fadeIn 0.3s ease-in-out;
    }

    .chat-input {
        display: flex;
        padding: 10px;
    }

    .chat-input input {
        flex-grow: 1;
        margin-right: 5px;
    }

    @keyframes fadeIn {
        from {
            opacity: 0;
        }
        to {
            opacity: 1;
        }
    }

    /* Mobile-specific styles */
    @media (max-width: 768px) {
        .controls {
            top: 60px;
            left: 0;
            right: 0;
            max-width: 100%;
            border-radius: 0;
        }

        .chat-container {
            left: 10px;
            right: 10px;
            width: auto;
        }
    }
</style>
