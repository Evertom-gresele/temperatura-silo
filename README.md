<body>
    <div id="body-debug-output">
        Mensagem de Debug do BODY: HTML carregado. Aguardando dados...
    </div>

    <h1>Receptor Universal de Dados FlutterFlow</h1>
    
    <p id="status-message">Aguardando dados do FlutterFlow...</p>

    <div class="data-display-section">
        <h2>Conteúdo Recebido:</h2>
        <pre id="received-data-output">Nenhum dado recebido ainda.</pre>
        <p>Tipo do dado original: <span id="data-type-output">Desconhecido</span></p>
        <p>Fonte do último dado: <span id="data-source-output">N/A</span></p>
    </div>

    <div class="log-section">
        <h2>Log de Execução JavaScript</h2>
        <ul id="execution-log">
            <li>Iniciando log de execução...</li>
        </ul>
    </div>

    <script>
        // ... (funções addLog, escapeHtml, updateAndDisplayData, displayReceivedData como antes) ...

        // =====================================================================
        // === FUNÇÃO EXISTENTE: Que você já está usando e será mantida ===
        // =====================================================================
        window.updateGraphData = function(data) {
            addLog('Função `updateGraphData` chamada!', 'step');
            addLog(`Tipo de 'data' recebido via updateGraphData: ${typeof data}.`, 'info');
            let dataSample;
            if (typeof data === 'object' && data !== null) {
                try {
                    dataSample = JSON.stringify(data).substring(0, Math.min(JSON.stringify(data).length, 200)) + '...';
                } catch (e) {
                    dataSample = String(data).substring(0, Math.min(String(data).length, 200)) + '...';
                }
            } else {
                dataSample = String(data).substring(0, Math.min(String(data).length, 200)) + '...';
            }
            addLog(`Amostra do conteúdo (updateGraphData): ${dataSample}`, 'info');

            updateAndDisplayData(data, "updateGraphData"); 
        };

        // =====================================================================
        // === NOVO: Listener para window.postMessage ===
        // =====================================================================
        window.addEventListener('message', function(event) {
            addLog('Evento `message` (window.postMessage) recebido!', 'step');
            addLog(`Origem da mensagem: ${event.origin}`, 'info'); 
            
            const receivedData = event.data; // O dado enviado via postMessage
            addLog(`Tipo de 'data' recebido via postMessage: ${typeof receivedData}.`, 'info');

            let dataSample;
            if (typeof receivedData === 'object' && receivedData !== null) {
                try {
                    dataSample = JSON.stringify(receivedData).substring(0, Math.min(JSON.stringify(receivedData).length, 200)) + '...';
                } catch (e) {
                    dataSample = String(receivedData).substring(0, Math.min(String(receivedData).length, 200)) + '...';
                }
            } else {
                dataSample = String(receivedData).substring(0, Math.min(String(receivedData).length, 200)) + '...';
            }
            addLog(`Amostra do conteúdo (postMessage): ${dataSample}`, 'info');

            updateAndDisplayData(receivedData, "postMessage");
        });

        // ... (DOMContentLoaded e beforeunload como antes) ...
    </script>
</body>
</html>
