<!DOCTYPE 2 html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Dados do Silo - Debug Detalhado</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 20px;
            background-color: #f4f4f4;
            color: #333;
        }
        h1 {
            color: #0056b3;
        }
        pre {
            background-color: #e9e9e9;
            padding: 15px;
            border-radius: 5px;
            overflow-x: auto;
        }
        .data-section {
            margin-bottom: 15px;
            border: 1px solid #ccc;
            padding: 10px;
            border-radius: 5px;
        }
        .data-section h2 {
            margin-top: 0;
            color: #007bff;
        }
        #status-message {
            font-weight: bold;
            color: #d63384;
        }
        .error-message {
            color: #dc3545;
            font-weight: bold;
        }
        .test-output {
            margin-top: 20px;
            padding: 10px;
            border: 2px dashed orange;
            background-color: #fffacd;
        }
        #execution-log {
            list-style-type: none;
            padding: 0;
            margin-top: 20px;
            border-top: 1px solid #ccc;
            padding-top: 10px;
        }
        #execution-log li {
            padding: 5px 0;
            border-bottom: 1px dotted #eee;
            font-size: 0.9em;
        }
        .log-success { color: #28a745; }
        .log-warn { color: #ffc107; }
        .log-error { color: #dc3545; }
        .log-info { color: #17a2b8; }
        .log-step { color: #6c757d; }
    </style>
</head>
<body>
    <h1>Dados Recebidos do FlutterFlow</h1>
    <p id="status-message">Aguardando dados... (Verifique o console para logs detalhados)</p>

    <div id="data-display">
        </div>

    <div class="test-output">
        <h2>Teste de Acesso ao DOM (Este deve aparecer!)</h2>
        <p>Se você vir esta mensagem, o HTML está sendo carregado corretamente.</p>
        <div id="runtime-content"></div>
    </div>

    <h2>Log de Execução JavaScript</h2>
    <ul id="execution-log">
        <li>Iniciando log de execução...</li>
    </ul>

    <script>
        let latestReceivedData = null; 
        let renderAttemptInterval = null; 
        let domAccessAttempts = 0; 
        let logCounter = 0; // Para numerar as etapas no log

        // Funções auxiliares para logar no console e no HTML
        function addLog(message, type = 'info') {
            logCounter++;
            const logList = document.getElementById('execution-log');
            if (logList) {
                const listItem = document.createElement('li');
                listItem.textContent = `${logCounter}. ${message}`;
                listItem.className = `log-${type}`;
                logList.appendChild(listItem);
                logList.scrollTop = logList.scrollHeight; // Scroll to bottom
            }
            // Logar no console também
            switch (type) {
                case 'success': console.log(`[SUCCESS] ${logCounter}. ${message}`); break;
                case 'warn': console.warn(`[WARN] ${logCounter}. ${message}`); break;
                case 'error': console.error(`[ERROR] ${logCounter}. ${message}`); break;
                case 'step': console.log(`[STEP] ${logCounter}. ${message}`); break;
                default: console.log(`[INFO] ${logCounter}. ${message}`); break;
            }
        }

        function escapeHtml(text) {
            const map = { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#039;' };
            return text.replace(/[&<>"']/g, function(m) { return map[m]; });
        }

        function attemptToDisplayData() {
            domAccessAttempts++;
            addLog(`Executando attemptToDisplayData. Tentativa: ${domAccessAttempts}.`, 'step');

            // 1. Tentar acessar elementos DOM
            const dataDisplayDiv = document.querySelector('#data-display');
            const statusMessage = document.querySelector('#status-message');
            const runtimeContentDiv = document.querySelector('#runtime-content');

            // 2. Logar o conteúdo do body para inspeção
            if (document.body) {
                addLog(`Conteúdo do document.body na tentativa ${domAccessAttempts}: ${document.body.innerHTML.
