<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Debug Extremo de Dados (com PostMessage)</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 20px;
            background-color: #f0f0f0; /* Cor de fundo inicial para teste */
            color: #333;
            display: flex;
            flex-direction: column;
            min-height: 95vh; /* Garante que o body ocupe a altura total */
        }
        h1 {
            color: #0056b3;
            text-align: center;
        }
        pre {
            background-color: #e9e9e9;
            padding: 15px;
            border-radius: 5px;
            overflow-x: auto;
            white-space: pre-wrap;
            word-wrap: break-word;
            border: 1px solid #ccc;
            margin-top: 10px;
            flex-grow: 1; /* Permite que o pre ocupe o espaço disponível */
        }
        #status-message {
            font-weight: bold;
            color: #d63384;
            text-align: center;
            margin-top: 15px;
            padding: 5px;
            background-color: #fff;
            border-radius: 5px;
        }
        .data-display-section {
            background-color: #ffffff;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
            margin-top: 20px;
            flex-grow: 1; /* Permite que esta seção cresça */
            display: flex; /* Para controlar o pre dentro dela */
            flex-direction: column;
        }
        .error-message {
            color: #dc3545;
            font-weight: bold;
        }
        .log-section {
            margin-top: 30px;
            border-top: 1px dashed #ccc;
            padding-top: 15px;
            flex-shrink: 0; /* Não encolhe */
        }
        #execution-log {
            list-style-type: none;
            padding: 0;
            max-height: 200px;
            overflow-y: auto;
            border: 1px solid #eee;
            padding: 10px;
            background-color: #f9f9f9;
            border-radius: 5px;
        }
        #execution-log li {
            padding: 5px 0;
            border-bottom: 1px dotted #eee;
            font-size: 0.9em;
            color: #555;
        }
        #execution-log li:last-child {
            border-bottom: none;
        }
        .log-success { color: #28a745; }
        .log-warn { color: #ffc107; }
        .log-error { color: #dc3545; }
        .log-info { color: #17a2b8; }
        .log-step { color: #6c757d; }

        /* Estilo para a caixa de debug direto no BODY */
        #body-debug-output {
            background-color: #ffe0b2; /* Laranja claro */
            border: 2px dashed #ff9800; /* Borda tracejada laranja */
            padding: 10px;
            margin-top: 20px;
            text-align: center;
            font-size: 1.1em;
            font-weight: bold;
            color: #e65100;
        }
    </style>
</head>
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
        // Variáveis globais
        let latestReceivedData = null; // Armazena o dado bruto recebido do FlutterFlow
        let latestDataSource = "N/A"; // Fonte do último dado (e.g., "updateGraphData" ou "postMessage")
        let dataPollingInterval = null; // ID do setInterval para a busca/exibição contínua
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
                logList.scrollTop = logList.scrollHeight; // Auto-scroll para o final
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

        // Função para escapar HTML para exibição segura
        function escapeHtml(text) {
            const map = { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#039;' };
            const stringText = String(text); 
            return stringText.replace(/[&<>"']/g, function(m) { return map[m]; });
        }

        // Função para atualizar as variáveis de dado e fonte, e disparar exibição
        function updateAndDisplayData(data, source) {
            addLog(`Dado recebido da fonte: ${source}`, 'step');
            latestReceivedData = data;
            latestDataSource = source;
            displayReceivedData(); // Chama a função de exibição imediatamente
        }

        // Função principal para exibir os dados no DOM
        function displayReceivedData() {
            addLog(`Tentando exibir dados. Estado do dado: ${latestReceivedData !== null ? 'Recebido' : 'Aguardando'}`, 'step');

            const statusMessage = document.getElementById('status-message');
            const receivedDataOutput = document.getElementById('received-data-output');
            const dataTypeOutput = document.getElementById('data-type-output');
            const dataSourceOutput = document.getElementById('data-source-output');
            const bodyDebugOutput = document.getElementById('body-debug-output');

            // Verifica se os elementos do DOM estão prontos
            if (!statusMessage || !receivedDataOutput || !dataTypeOutput || !dataSourceOutput || !bodyDebugOutput) {
                addLog('Elementos do DOM não encontrados. Re-tentando na próxima checagem.', 'warn');
                return;
            }

            // Atualiza a mensagem de debug do BODY em cada tentativa
            if (latestReceivedData === null) {
                bodyDebugOutput.textContent = 'Mensagem de Debug do BODY: Aguardando dados...';
                bodyDebugOutput.style.backgroundColor = '#ffe0b2'; // Laranja claro
            } else {
                bodyDebugOutput.textContent = 'Mensagem de Debug do BODY: Dados Recebidos!';
                bodyDebugOutput.style.backgroundColor = '#d4edda'; // Verde claro para indicar sucesso
            }


            if (latestReceivedData !== null) {
                addLog('Dado encontrado na variável global `latestReceivedData`. Processando para exibição.', 'success');
                statusMessage.textContent = 'Último Dado Recebido:';

                let contentToDisplay;
                let originalType = typeof latestReceivedData;
                dataTypeOutput.textContent = originalType;
                dataSourceOutput.textContent = latestDataSource; // Exibe a fonte do dado

                // Tenta stringificar o dado para JSON formatado se for um objeto, senão exibe como string
                if (originalType === 'object' && latestReceivedData !== null) {
                    try {
                        contentToDisplay = JSON.stringify(latestReceivedData, null, 2);
                        addLog('Dado é um Objeto JS válido. Stringificado para JSON formatado.', 'info');
                    } catch (e) {
                        contentToDisplay = String(latestReceivedData); // Fallback para objetos complexos ou circulares
                        addLog(`Erro ao stringificar objeto para JSON: ${e.message}. Exibindo como string bruta.`, 'error');
                        dataTypeOutput.textContent += ' (Erro JSON.stringify)';
                    }
                } else {
                    contentToDisplay = String(latestReceivedData); // Exibe qualquer outro tipo como string bruta
                    addLog(`Dado é do tipo '${originalType}'. Exibindo como string bruta.`, 'info');
                }
                
                receivedDataOutput.innerHTML = escapeHtml(contentToDisplay);
                addLog('Conteúdo adicionado ao elemento de exibição.', 'success');

            } else {
                addLog('Nenhum dado ainda em `latestReceivedData`. Mantendo mensagem de aguardo.', 'info');
                statusMessage.textContent = 'Aguardando dados do FlutterFlow...';
                receivedDataOutput.innerHTML = 'Nenhum dado recebido ainda.';
                dataTypeOutput.textContent = 'Desconhecido';
                dataSourceOutput.textContent = 'N/A';
            }
        }

        // =====================================================================
        // === FUNÇÃO EXISTENTE: Que você já está usando e será mantida ===
        // =====================================================================
        window.updateGraphData = function(data) {
            addLog('Função `updateGraphData` chamada!', 'step');
            
            // Loga o tipo e uma amostra do conteúdo para o console.
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
            addLog(`Origem da mensagem: ${event.origin}`, 'info'); // Importante para segurança
            
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

            // Atualiza e exibe os dados usando a função genérica
            updateAndDisplayData(receivedData, "postMessage");
        });


        // Quando o documento HTML estiver completamente carregado
        document.addEventListener('DOMContentLoaded', () => {
            addLog('Evento DOMContentLoaded disparado. HTML e DOM estão prontos.', 'success');
            
            // Inicia a "busca" contínua a cada 3 segundos.
            // Esta função verificará e exibirá os dados se disponíveis.
            if (!dataPollingInterval) {
                dataPollingInterval = setInterval(displayReceivedData, 3000); // Tenta a cada 3 segundos
                addLog('Intervalo de busca/exibição de dados (3 segundos) iniciado.', 'info');
            }

            // Uma chamada inicial para garantir que o estado inicial do DOM seja exibido.
            displayReceivedData(); 
        });

        // Limpa o intervalo quando a página está sendo descarregada para evitar vazamentos
        window.addEventListener('beforeunload', () => {
            addLog('Evento `beforeunload` disparado. Limpando intervalo de busca.', 'info');
            if (dataPollingInterval) {
                clearInterval(dataPollingInterval);
                dataPollingInterval = null;
            }
        });

        addLog('Script index3.html carregado e pronto para inicialização.', 'success');
    </script>
</body>
</html>
