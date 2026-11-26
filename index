<?php
// Your Google Drive file ID
$defaultFileId = '1qOMbyhkbt_basf53QRQ1hhFwIa0UdqNp';
$fileId = isset($_GET['id']) && !empty($_GET['id'])
    ? preg_replace('/[^a-zA-Z0-9_-]/', '', $_GET['id'])
    : $defaultFileId;

// Stream the video if ?stream=1 is set; otherwise render the HTML player
if (isset($_GET['stream']) && $_GET['stream'] == '1') {
    if (empty($fileId)) {
        http_response_code(400);
        exit('File ID missing');
    }

    @set_time_limit(0);

    // Disable output buffering so streaming stays smooth
    if (function_exists('apache_setenv')) {
        @apache_setenv('no-gzip', 1);
    }
    @ini_set('zlib.output_compression', 0);
    @ini_set('implicit_flush', 1);
    while (ob_get_level() > 0) {
        ob_end_flush();
    }
    ob_implicit_flush(1);

    $baseHeaders = [
        "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36",
        "Accept: */*",
        "Connection: keep-alive"
    ];

    $rangeHeader = isset($_SERVER['HTTP_RANGE']) ? ["Range: " . $_SERVER['HTTP_RANGE']] : [];

    // Probe a small byte range first to see if Google Drive returns the file directly or an HTML confirmation page
    $probeUrl = "https://drive.usercontent.google.com/download?id=" . $fileId . "&export=download";
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => $probeUrl,
        CURLOPT_HTTPHEADER => array_merge($baseHeaders, ["Range: bytes=0-65535"]),
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_FOLLOWLOCATION => true,
        CURLOPT_HEADER => true,
        CURLOPT_SSL_VERIFYPEER => true,
        CURLOPT_TIMEOUT => 30,
    ]);
    $probeResponse = curl_exec($ch);
    $headerSize = curl_getinfo($ch, CURLINFO_HEADER_SIZE);
    $effectiveUrl = curl_getinfo($ch, CURLINFO_EFFECTIVE_URL);
    curl_close($ch);

    $rawHeaders = substr($probeResponse, 0, $headerSize);
    $bodySnippet = substr($probeResponse, $headerSize);
    $streamUrl = $effectiveUrl ?: $probeUrl;

    // For large files, extract form action and hidden inputs if Google Drive returns an HTML download warning page
    if (stripos($rawHeaders, 'text/html') !== false || stripos($bodySnippet, '<!doctype html') !== false || stripos($bodySnippet, '<html') !== false) {
        if (preg_match('/action="([^"]+)"/i', $bodySnippet, $actionMatch)) {
            $formAction = html_entity_decode($actionMatch[1], ENT_QUOTES, 'UTF-8');
            $queryParams = [];

            if (preg_match_all('/<input[^>]*name="([^"]+)"[^>]*value="([^"]*)"/i', $bodySnippet, $inputs, PREG_SET_ORDER)) {
                foreach ($inputs as $inp) {
                    $queryParams[$inp[1]] = html_entity_decode($inp[2], ENT_QUOTES, 'UTF-8');
                }
            }

            if (!isset($queryParams['id'])) {
                $queryParams['id'] = $fileId;
            }
            if (!isset($queryParams['export'])) {
                $queryParams['export'] = 'download';
            }
            if (!isset($queryParams['confirm'])) {
                $queryParams['confirm'] = 't';
            }

            $sep = (strpos($formAction, '?') === false) ? '?' : '&';
            $streamUrl = $formAction . $sep . http_build_query($queryParams);
        } else {
            $streamUrl = "https://drive.usercontent.google.com/download?id=" . $fileId . "&export=download&confirm=t";
        }
    }

    // Start streaming the actual video
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => $streamUrl,
        CURLOPT_HTTPHEADER => array_merge($baseHeaders, $rangeHeader),
        CURLOPT_FOLLOWLOCATION => true,
        CURLOPT_BUFFERSIZE => 1024 * 1024, // 1MB buffer
        CURLOPT_RETURNTRANSFER => false,
        CURLOPT_SSL_VERIFYPEER => true,
        CURLOPT_TIMEOUT => 0,
        CURLOPT_HEADERFUNCTION => function ($curl, $header) {
            $len = strlen($header);
            $trimmed = trim($header);
            if ($trimmed === '') {
                return $len;
            }

            if (stripos($trimmed, 'HTTP/') === 0) {
                header($trimmed, true);
                return $len;
            }

            $parts = explode(':', $trimmed, 2);
            if (count($parts) === 2) {
                $name = strtolower(trim($parts[0]));
                $val = trim($parts[1]);
                $allowed = ['content-type', 'content-length', 'content-range', 'accept-ranges'];
                if (in_array($name, $allowed, true)) {
                    // Prevent an HTML error page from being sent as video/mp4
                    if ($name === 'content-type' && stripos($val, 'text/html') !== false) {
                        http_response_code(502);
                        return 0;
                    }
                    header($name . ': ' . $val, true);
                }
            }
            return $len;
        },
        CURLOPT_WRITEFUNCTION => function ($curl, $data) {
            if (connection_aborted()) {
                return 0;
            }
            echo $data;
            @ob_flush();
            @flush();
            return strlen($data);
        }
    ]);

    curl_exec($ch);
    curl_close($ch);
    exit;
}

$videoSrc = htmlspecialchars($_SERVER['PHP_SELF'] . '?id=' . urlencode($fileId) . '&stream=1', ENT_QUOTES, 'UTF-8');
?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="mobile-web-app-capable" content="yes">
    <title>Full Page Stream Player</title>
    <style>
        :root {
            --accent: #6366f1;
            --accent-hover: #818cf8;
            --bg-dark: #050507;
            --text-main: #f4f4f5;
            --text-muted: #a1a1aa;
            --touch-target: 44px;
        }

        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
            user-select: none;
            -webkit-user-select: none;
            -webkit-tap-highlight-color: transparent;
        }

        html, body {
            width: 100%;
            height: 100%;
            height: 100dvh;
            overflow: hidden;
            overscroll-behavior: none;
            background: var(--bg-dark);
            color: var(--text-main);
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
        }

        .player-container {
            position: relative;
            width: 100vw;
            height: 100dvh;
            background: #000;
            display: flex;
            align-items: center;
            justify-content: center;
            overflow: hidden;
            touch-action: manipulation;
        }

        video {
            width: 100%;
            height: 100%;
            object-fit: contain;
            cursor: pointer;
            background: #000;
            display: block;
        }

        /* Loading Spinner + Status */
        .loader-wrap {
            position: absolute;
            inset: 0;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            gap: 12px;
            padding: 20px;
            background: rgba(0, 0, 0, 0.4);
            pointer-events: none;
            opacity: 0;
            transition: opacity 0.2s ease;
            z-index: 10;
        }

        .player-container.loading .loader-wrap {
            opacity: 1;
        }

        .loader {
            width: clamp(44px, 10vw, 56px);
            height: clamp(44px, 10vw, 56px);
            border: 4px solid rgba(255, 255, 255, 0.16);
            border-top-color: var(--accent);
            border-radius: 50%;
            animation: spin 0.85s linear infinite;
        }

        .loader-text {
            font-size: clamp(12px, 3.2vw, 14px);
            color: var(--text-main);
            text-align: center;
            max-width: 90%;
            background: rgba(10, 10, 14, 0.82);
            padding: 6px 14px;
            border-radius: 999px;
        }

        @keyframes spin {
            to { transform: rotate(360deg); }
        }

        /* Center Play Button */
        .center-play {
            position: absolute;
            width: 76px;
            height: 76px;
            border-radius: 50%;
            background: rgba(99, 102, 241, 0.92);
            backdrop-filter: blur(8px);
            border: none;
            color: #fff;
            display: flex;
            align-items: center;
            justify-content: center;
            cursor: pointer;
            box-shadow: 0 12px 32px rgba(99, 102, 241, 0.5);
            transition: transform 0.2s ease, opacity 0.25s ease, background 0.2s ease;
            z-index: 9;
        }

        .center-play:hover {
            transform: scale(1.08);
            background: var(--accent-hover);
        }

        .center-play:active {
            transform: scale(0.95);
        }

        .player-container.playing .center-play,
        .player-container.loading .center-play {
            opacity: 0;
            pointer-events: none;
            transform: scale(0.9);
        }

        /* Skip Feedback Toast */
        .skip-badge {
            position: absolute;
            top: 50%;
            transform: translateY(-50%);
            background: rgba(0, 0, 0, 0.78);
            backdrop-filter: blur(6px);
            padding: 10px 18px;
            border-radius: 999px;
            font-size: 14px;
            font-weight: 600;
            opacity: 0;
            pointer-events: none;
            transition: opacity 0.2s ease;
            z-index: 12;
        }

        .skip-badge.left { left: clamp(16px, 8%, 10%); }
        .skip-badge.right { right: clamp(16px, 8%, 10%); }
        .skip-badge.show { opacity: 1; }

        /* Controls Overlay */
        .controls-wrapper {
            position: absolute;
            bottom: 0;
            left: 0;
            right: 0;
            background: linear-gradient(to top, rgba(0, 0, 0, 0.94) 0%, rgba(0, 0, 0, 0.62) 65%, transparent 100%);
            padding: 36px max(12px, calc(12px + env(safe-area-inset-left))) max(12px, calc(10px + env(safe-area-inset-bottom))) max(12px, calc(12px + env(safe-area-inset-right)));
            display: flex;
            flex-direction: column;
            gap: 10px;
            opacity: 1;
            transition: opacity 0.28s ease, transform 0.28s ease;
            z-index: 15;
        }

        .player-container.playing.hide-controls .controls-wrapper {
            opacity: 0;
            transform: translateY(10px);
            pointer-events: none;
        }

        .player-container.playing.hide-controls {
            cursor: none;
        }

        /* Progress Bar with Generous Touch Hitbox */
        .progress-area {
            position: relative;
            width: 100%;
            height: 24px;
            display: flex;
            align-items: center;
            cursor: pointer;
            touch-action: none;
        }

        .progress-track {
            position: relative;
            width: 100%;
            height: 6px;
            background: rgba(255, 255, 255, 0.24);
            border-radius: 99px;
            transition: height 0.15s ease;
            overflow: visible;
        }

        .progress-area:hover .progress-track,
        .progress-area.scrubbing .progress-track {
            height: 8px;
        }

        .buffered-bar {
            position: absolute;
            top: 0;
            left: 0;
            height: 100%;
            background: rgba(255, 255, 255, 0.36);
            border-radius: 99px;
            width: 0%;
            pointer-events: none;
        }

        .progress-bar {
            position: relative;
            height: 100%;
            background: var(--accent);
            border-radius: 99px;
            width: 0%;
            pointer-events: none;
        }

        .progress-bar::after {
            content: "";
            position: absolute;
            right: -7px;
            top: 50%;
            transform: translateY(-50%) scale(0);
            width: 15px;
            height: 15px;
            background: #fff;
            border-radius: 50%;
            box-shadow: 0 0 8px rgba(0, 0, 0, 0.55);
            transition: transform 0.15s ease;
        }

        .progress-area:hover .progress-bar::after,
        .progress-area.scrubbing .progress-bar::after {
            transform: translateY(-50%) scale(1);
        }

        /* Bottom Bar Buttons */
        .controls-row {
            display: flex;
            align-items: center;
            justify-content: space-between;
            gap: 8px;
            min-height: var(--touch-target);
        }

        .controls-left, .controls-right {
            display: flex;
            align-items: center;
            gap: clamp(4px, 1.2vw, 12px);
        }

        .btn {
            background: none;
            border: none;
            color: var(--text-main);
            cursor: pointer;
            min-width: var(--touch-target);
            min-height: var(--touch-target);
            display: inline-flex;
            align-items: center;
            justify-content: center;
            padding: 8px;
            border-radius: 10px;
            transition: background 0.15s ease, color 0.15s ease, transform 0.1s ease;
        }

        .btn:hover {
            background: rgba(255, 255, 255, 0.14);
            color: #fff;
        }

        .btn:active {
            transform: scale(0.92);
            background: rgba(255, 255, 255, 0.2);
        }

        .btn svg {
            width: 22px;
            height: 22px;
            fill: currentColor;
            flex-shrink: 0;
        }

        /* Volume Slider */
        .volume-group {
            display: flex;
            align-items: center;
            gap: 4px;
        }

        .volume-slider {
            width: 78px;
            height: 24px;
            accent-color: var(--accent);
            cursor: pointer;
            touch-action: none;
        }

        .time-display {
            font-size: clamp(11.5px, 2.9vw, 14px);
            font-variant-numeric: tabular-nums;
            color: var(--text-muted);
            white-space: nowrap;
            padding-left: 2px;
        }

        .time-display span {
            color: var(--text-main);
        }

        /* Speed Selector */
        .speed-select {
            background: rgba(255, 255, 255, 0.13);
            color: var(--text-main);
            border: 1px solid rgba(255, 255, 255, 0.2);
            border-radius: 8px;
            min-height: 36px;
            padding: 0 8px;
            font-size: 12px;
            font-weight: 600;
            cursor: pointer;
            outline: none;
        }

        .speed-select option {
            background: #18181b;
            color: #fff;
        }

        /* Phones (Portrait & Narrow Screens) */
        @media (max-width: 640px) {
            .controls-wrapper {
                padding-top: 28px;
                gap: 6px;
            }

            .volume-slider,
            .pip-btn {
                display: none;
            }

            .controls-left,
            .controls-right {
                gap: 2px;
            }

            .btn {
                min-width: 38px;
                min-height: 40px;
                padding: 6px;
            }

            .btn svg {
                width: 20px;
                height: 20px;
            }

            .center-play {
                width: 64px;
                height: 64px;
            }

            .speed-select {
                min-height: 34px;
                padding: 0 6px;
                font-size: 11.5px;
            }

            .progress-bar::after {
                transform: translateY(-50%) scale(0.85);
            }
        }

        /* Very Small Phones (e.g., 320px - 375px) */
        @media (max-width: 375px) {
            #rewindBtn,
            #forwardBtn {
                display: none;
            }

            .time-display {
                font-size: 11px;
            }
        }

        /* Landscape Mobile Phones */
        @media (max-height: 500px) and (orientation: landscape) {
            .controls-wrapper {
                padding: 22px max(14px, env(safe-area-inset-left)) max(8px, env(safe-area-inset-bottom)) max(14px, env(safe-area-inset-right));
                gap: 6px;
            }

            .center-play {
                width: 56px;
                height: 56px;
            }

            .progress-area {
                height: 20px;
            }

            .btn {
                min-height: 36px;
                min-width: 38px;
            }
        }
    </style>
</head>
<body>

    <div class="player-container loading" id="playerContainer">
        <video id="video" preload="auto" playsinline webkit-playsinline>
            <source src="<?= $videoSrc ?>" type="video/mp4">
            Your browser does not support the HTML5 video player.
        </video>

        <div class="loader-wrap" id="loaderWrap">
            <div class="loader"></div>
            <div class="loader-text" id="loaderText">Loading...</div>
        </div>

        <button class="center-play" id="centerPlayBtn" aria-label="Play Video">
            <svg viewBox="0 0 24 24" width="34" height="34" fill="currentColor">
                <path d="M8 5v14l11-7z"/>
            </svg>
        </button>

        <div class="skip-badge left" id="skipLeft">-10s</div>
        <div class="skip-badge right" id="skipRight">+10s</div>

        <div class="controls-wrapper" id="controls">
            <div class="progress-area" id="progressArea" aria-label="Seek timeline">
                <div class="progress-track">
                    <div class="buffered-bar" id="bufferedBar"></div>
                    <div class="progress-bar" id="progressBar"></div>
                </div>
            </div>

            <div class="controls-row">
                <div class="controls-left">
                    <button class="btn" id="playPauseBtn" title="Play/Pause (Space)" aria-label="Play or Pause">
                        <svg id="playIcon" viewBox="0 0 24 24"><path d="M8 5v14l11-7z"/></svg>
                    </button>

                    <button class="btn" id="rewindBtn" title="Back 10s" aria-label="Rewind 10 seconds">
                        <svg viewBox="0 0 24 24"><path d="M11 18V6l-8.5 6 8.5 6zm.5-6l8.5 6V6l-8.5 6z"/></svg>
                    </button>

                    <button class="btn" id="forwardBtn" title="Forward 10s" aria-label="Forward 10 seconds">
                        <svg viewBox="0 0 24 24"><path d="M4 18l8.5-6L4 6v12zm9-12v12l8.5-6L13 6z"/></svg>
                    </button>

                    <div class="volume-group">
                        <button class="btn" id="muteBtn" title="Mute (M)" aria-label="Mute or Unmute">
                            <svg id="volIcon" viewBox="0 0 24 24"><path d="M3 9v6h4l5 5V4L7 9H3zm13.5 3c0-1.77-1.02-3.29-2.5-4.03v8.05c1.48-.73 2.5-2.25 2.5-4.02zM14 3.23v2.06c2.89.86 5 3.54 5 6.71s-2.11 5.85-5 6.71v2.06c4.01-.91 7-4.49 7-8.77s-2.99-7.86-7-8.77z"/></svg>
                        </button>
                        <input type="range" class="volume-slider" id="volumeSlider" min="0" max="1" step="0.05" value="1" aria-label="Volume">
                    </div>

                    <div class="time-display">
                        <span id="currentTime">00:00</span> / <span id="duration">00:00</span>
                    </div>
                </div>

                <div class="controls-right">
                    <select class="speed-select" id="speedSelect" title="Playback Speed" aria-label="Playback Speed">
                        <option value="0.5">0.5x</option>
                        <option value="0.75">0.75x</option>
                        <option value="1" selected>1x</option>
                        <option value="1.25">1.25x</option>
                        <option value="1.5">1.5x</option>
                        <option value="2">2x</option>
                    </select>

                    <button class="btn pip-btn" id="pipBtn" title="Picture-in-Picture" aria-label="Picture in Picture">
                        <svg viewBox="0 0 24 24"><path d="M19 7h-8v6h8V7zm2-4H3c-1.1 0-2 .9-2 2v14c0 1.1.9 1.98 2 1.98h18c1.1 0 2-.88 2-1.98V5c0-1.1-.9-2-2-2zm0 16.01H3V4.98h18v14.03z"/></svg>
                    </button>

                    <button class="btn" id="fullscreenBtn" title="Fullscreen (F)" aria-label="Toggle Fullscreen">
                        <svg id="fsIcon" viewBox="0 0 24 24"><path d="M7 14H5v5h5v-2H7v-3zm-2-4h2V7h3V5H5v5zm12 7h-3v2h5v-5h-2v3zM14 5v2h3v3h2V5h-5z"/></svg>
                    </button>
                </div>
            </div>
        </div>
    </div>

    <script>
        const container = document.getElementById('playerContainer');
        const video = document.getElementById('video');
        const loaderText = document.getElementById('loaderText');
        const playPauseBtn = document.getElementById('playPauseBtn');
        const centerPlayBtn = document.getElementById('centerPlayBtn');
        const playIcon = document.getElementById('playIcon');
        const rewindBtn = document.getElementById('rewindBtn');
        const forwardBtn = document.getElementById('forwardBtn');
        const muteBtn = document.getElementById('muteBtn');
        const volIcon = document.getElementById('volIcon');
        const volumeSlider = document.getElementById('volumeSlider');
        const progressArea = document.getElementById('progressArea');
        const progressBar = document.getElementById('progressBar');
        const bufferedBar = document.getElementById('bufferedBar');
        const currentTimeEl = document.getElementById('currentTime');
        const durationEl = document.getElementById('duration');
        const speedSelect = document.getElementById('speedSelect');
        const pipBtn = document.getElementById('pipBtn');
        const fullscreenBtn = document.getElementById('fullscreenBtn');
        const skipLeft = document.getElementById('skipLeft');
        const skipRight = document.getElementById('skipRight');

        let hideControlsTimeout;
        let lastTapTime = 0;
        let lastTapX = 0;
        let singleTapTimeout;

        // Sync dynamic viewport height on mobile browsers where 100dvh shifts with bars
        function syncViewportHeight() {
            const h = window.visualViewport ? window.visualViewport.height : window.innerHeight;
            container.style.height = `${h}px`;
        }
        syncViewportHeight();
        window.addEventListener('resize', syncViewportHeight);
        window.addEventListener('orientationchange', () => setTimeout(syncViewportHeight, 150));
        if (window.visualViewport) {
            window.visualViewport.addEventListener('resize', syncViewportHeight);
        }

        function formatTime(sec) {
            if (!isFinite(sec) || isNaN(sec)) return '00:00';
            const hrs = Math.floor(sec / 3600);
            const mins = Math.floor((sec % 3600) / 60);
            const secs = Math.floor(sec % 60);
            if (hrs > 0) {
                return `${hrs}:${String(mins).padStart(2, '0')}:${String(secs).padStart(2, '0')}`;
            }
            return `${String(mins).padStart(2, '0')}:${String(secs).padStart(2, '0')}`;
        }

        function togglePlay() {
            if (video.paused || video.ended) {
                video.play().catch(() => {});
            } else {
                video.pause();
            }
        }

        function updatePlayState() {
            if (video.paused) {
                container.classList.remove('playing', 'hide-controls');
                playIcon.innerHTML = '<path d="M8 5v14l11-7z"/>';
            } else {
                container.classList.add('playing');
                playIcon.innerHTML = '<path d="M6 19h4V5H6v14zm8-14v14h4V5h-4z"/>';
                resetControlsTimer();
            }
        }

        function skip(seconds) {
            if (!isFinite(video.duration)) return;
            video.currentTime = Math.max(0, Math.min(video.duration, video.currentTime + seconds));
            const badge = seconds < 0 ? skipLeft : skipRight;
            badge.classList.add('show');
            clearTimeout(badge.timer);
            badge.timer = setTimeout(() => badge.classList.remove('show'), 500);
        }

        function resetControlsTimer() {
            container.classList.remove('hide-controls');
            clearTimeout(hideControlsTimeout);
            if (!video.paused) {
                hideControlsTimeout = setTimeout(() => {
                    container.classList.add('hide-controls');
                }, 2800);
            }
        }

        // Play / Pause Events
        playPauseBtn.addEventListener('click', togglePlay);
        centerPlayBtn.addEventListener('click', togglePlay);

        // Desktop click vs. mobile double-tap left/right to skip -10s / +10s
        video.addEventListener('click', (e) => {
            if (window.matchMedia('(pointer: coarse)').matches) {
                const now = Date.now();
                const rect = video.getBoundingClientRect();
                const tapX = e.clientX - rect.left;
                const isDoubleTap = (now - lastTapTime < 300) && Math.abs(tapX - lastTapX) < rect.width * 0.25;

                clearTimeout(singleTapTimeout);

                if (isDoubleTap) {
                    lastTapTime = 0;
                    if (tapX < rect.width * 0.35) {
                        skip(-10);
                    } else if (tapX > rect.width * 0.65) {
                        skip(10);
                    } else {
                        togglePlay();
                    }
                    resetControlsTimer();
                } else {
                    lastTapTime = now;
                    lastTapX = tapX;
                    singleTapTimeout = setTimeout(() => {
                        if (video.paused) {
                            togglePlay();
                        } else if (container.classList.contains('hide-controls')) {
                            resetControlsTimer();
                        } else {
                            container.classList.add('hide-controls');
                        }
                    }, 260);
                }
            } else {
                togglePlay();
            }
        });

        video.addEventListener('play', updatePlayState);
        video.addEventListener('pause', updatePlayState);

        // Buffering / Loading States
        video.addEventListener('loadstart', () => {
            loaderText.textContent = 'Starting stream...';
            container.classList.add('loading');
        });
        video.addEventListener('waiting', () => {
            loaderText.textContent = 'Buffering...';
            container.classList.add('loading');
        });
        video.addEventListener('playing', () => container.classList.remove('loading'));
        video.addEventListener('canplay', () => container.classList.remove('loading'));
        video.addEventListener('error', () => {
            loaderText.textContent = 'Unable to load video (check if the file is public)';
            container.classList.add('loading');
        });

        // Timeline & Buffer Updates
        video.addEventListener('loadedmetadata', () => {
            durationEl.textContent = formatTime(video.duration);
        });

        video.addEventListener('timeupdate', () => {
            currentTimeEl.textContent = formatTime(video.currentTime);
            const pct = (video.currentTime / video.duration) * 100 || 0;
            progressBar.style.width = `${pct}%`;
        });

        video.addEventListener('progress', () => {
            if (video.buffered.length > 0 && video.duration > 0) {
                const bufferedEnd = video.buffered.end(video.buffered.length - 1);
                const pct = Math.min(100, (bufferedEnd / video.duration) * 100);
                bufferedBar.style.width = `${pct}%`;
            }
        });

        // Seek on Progress Bar Click/Drag
        let isScrubbing = false;
        function scrub(e) {
            if (!isFinite(video.duration) || video.duration <= 0) return;
            const rect = progressArea.getBoundingClientRect();
            const clientX = (e.touches && e.touches.length) ? e.touches[0].clientX : e.clientX;
            const pos = Math.max(0, Math.min(1, (clientX - rect.left) / rect.width));
            video.currentTime = pos * video.duration;
            progressBar.style.width = `${pos * 100}%`;
        }

        progressArea.addEventListener('mousedown', (e) => {
            isScrubbing = true;
            progressArea.classList.add('scrubbing');
            scrub(e);
        });
        window.addEventListener('mousemove', (e) => { if (isScrubbing) scrub(e); });
        window.addEventListener('mouseup', () => {
            if (isScrubbing) {
                isScrubbing = false;
                progressArea.classList.remove('scrubbing');
            }
        });

        progressArea.addEventListener('touchstart', (e) => {
            isScrubbing = true;
            progressArea.classList.add('scrubbing');
            resetControlsTimer();
            scrub(e);
        }, { passive: true });
        window.addEventListener('touchmove', (e) => { if (isScrubbing) scrub(e); }, { passive: true });
        window.addEventListener('touchend', () => {
            if (isScrubbing) {
                isScrubbing = false;
                progressArea.classList.remove('scrubbing');
            }
        });

        // Skip Buttons
        rewindBtn.addEventListener('click', () => skip(-10));
        forwardBtn.addEventListener('click', () => skip(10));

        // Volume & Mute
        function updateVolumeIcon() {
            if (video.muted || video.volume === 0) {
                volIcon.innerHTML = '<path d="M16.5 12c0-1.77-1.02-3.29-2.5-4.03v2.21l2.45 2.45c.03-.2.05-.41.05-.63zm2.5 0c0 .94-.2 1.82-.54 2.64l1.51 1.51C20.63 14.91 21 13.5 21 12c0-4.28-2.99-7.86-7-8.77v2.06c2.89.86 5 3.54 5 6.71zM4.27 3L3 4.27 7.73 9H3v6h4l5 5v-6.73l4.25 4.25c-.67.52-1.42.93-2.25 1.18v2.06c1.38-.31 2.63-.95 3.69-1.81L19.73 21 21 19.73l-9-9L4.27 3zM12 4L9.91 6.09 12 8.18V4z"/>';
            } else {
                volIcon.innerHTML = '<path d="M3 9v6h4l5 5V4L7 9H3zm13.5 3c0-1.77-1.02-3.29-2.5-4.03v8.05c1.48-.73 2.5-2.25 2.5-4.02zM14 3.23v2.06c2.89.86 5 3.54 5 6.71s-2.11 5.85-5 6.71v2.06c4.01-.91 7-4.49 7-8.77s-2.99-7.86-7-8.77z"/>';
            }
        }

        muteBtn.addEventListener('click', () => {
            video.muted = !video.muted;
            volumeSlider.value = video.muted ? 0 : video.volume;
            updateVolumeIcon();
        });

        volumeSlider.addEventListener('input', (e) => {
            video.volume = e.target.value;
            video.muted = video.volume === 0;
            updateVolumeIcon();
        });

        // Playback Speed
        speedSelect.addEventListener('change', (e) => {
            video.playbackRate = parseFloat(e.target.value);
        });

        // Picture-in-Picture (Standard + iOS Safari WebKit fallback)
        pipBtn.addEventListener('click', async () => {
            try {
                if (document.pictureInPictureElement) {
                    await document.exitPictureInPicture();
                } else if (document.pictureInPictureEnabled && video.requestPictureInPicture) {
                    await video.requestPictureInPicture();
                } else if (typeof video.webkitSetPresentationMode === 'function') {
                    const mode = video.webkitPresentationMode === 'picture-in-picture' ? 'inline' : 'picture-in-picture';
                    video.webkitSetPresentationMode(mode);
                }
            } catch (_) {}
        });

        // Fullscreen (Standard + iOS Safari video element fallback)
        function isFullscreen() {
            return Boolean(document.fullscreenElement || document.webkitFullscreenElement || video.webkitDisplayingFullscreen);
        }

        async function toggleFullscreen() {
            try {
                if (!isFullscreen()) {
                    if (container.requestFullscreen) {
                        await container.requestFullscreen();
                    } else if (container.webkitRequestFullscreen) {
                        container.webkitRequestFullscreen();
                    } else if (typeof video.webkitEnterFullscreen === 'function') {
                        video.webkitEnterFullscreen();
                    }
                } else {
                    if (document.exitFullscreen) {
                        await document.exitFullscreen();
                    } else if (document.webkitExitFullscreen) {
                        document.webkitExitFullscreen();
                    }
                }
            } catch (_) {}
            setTimeout(syncViewportHeight, 150);
        }

        fullscreenBtn.addEventListener('click', toggleFullscreen);
        video.addEventListener('dblclick', toggleFullscreen);

        // Auto-hide controls on mouse move / touch
        container.addEventListener('mousemove', resetControlsTimer);
        container.addEventListener('touchstart', resetControlsTimer, { passive: true });

        // Keyboard Shortcuts
        document.addEventListener('keydown', (e) => {
            if (['INPUT', 'SELECT'].includes(document.activeElement.tagName)) return;
            switch (e.key.toLowerCase()) {
                case ' ':
                case 'k':
                    e.preventDefault();
                    togglePlay();
                    break;
                case 'arrowleft':
                    skip(-10);
                    break;
                case 'arrowright':
                    skip(10);
                    break;
                case 'm':
                    muteBtn.click();
                    break;
                case 'f':
                    toggleFullscreen();
                    break;
            }
        });
    </script>
</body>
</html>
