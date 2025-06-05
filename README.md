<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Receptor Universal de Dados FlutterFlow</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 20px;
            background-color: #f4f4f4;
            color: #333;
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
            white-space: pre-wrap; /* Garante quebras de linha para JSON formatado */
            word-wrap: break-word;  /* Quebra palavras longas se necessário */
            border: 1px solid #ccc;
            margin-top: 10px;
        }
        #status-message {
            font-weight: bold;
            color: #d63384;
            text-align: center;
            margin-top: 15px;
        }
        .data-display-section {
            background-color: #ffffff;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
            margin-top: 20px;
        }
        .error-message {
            color: #dc3545;
            font-weight: bold;
        }
        .log-section {
            margin-top: 30px;
            border-top: 1px dashed #ccc;
            padding-top: 15px;
        }
        #execution-log {
            list-style-type: none;
            padding: 0;
            max-height: 200px; /* Limita a altura para o log não ocupar a tela toda */
            overflow-y: auto; /* Adiciona scroll se o log for muito grande */
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
    </style>
</head>
<body>
    <h1>Receptor Universal de Dados FlutterFlow</h1>
    
    <p id="status-message">Aguardando dados do FlutterFlow...</p>

    <div class="data-display-section">
        <h2>Conteúdo Recebido:</h2>
        <pre id="received-data-output">Nenhum dado recebido ainda.</pre>
        <p>Tipo do dado original: <span id="data-type-output">Desconhecido</span></p>
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

        // Função principal para exibir os dados no DOM
        function displayReceivedData() {
            addLog(`Tentando exibir dados. Estado do dado: ${latestReceivedData !== null ? 'Recebido' : 'Aguardando'}`, 'step');

            const statusMessage = document.getElementById('status-message');
            const receivedDataOutput = document.getElementById('received-data-output');
            const dataTypeOutput = document.getElementById('data-type-output');

            // Verifica se os elementos do DOM estão prontos
            if (!statusMessage || !receivedDataOutput || !dataTypeOutput) {
                addLog('Elementos do DOM não encontrados. Re-tentando na próxima checagem.', 'warn');
                return;
            }

            if (latestReceivedData !== null) {
                addLog('Dado encontrado na variável global `latestReceivedData`. Processando para exibição.', 'success');
                statusMessage.textContent = 'Último Dado Recebido:';

                let contentToDisplay;
                let originalType = typeof latestReceivedData;
                dataTypeOutput.textContent = originalType;

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
            }
        }

        // Esta é a função que o FlutterFlow (ou o ambiente que você está usando) chamará
        // para enviar dados para o WebView.
        window.updateGraphData = function(data) {
            addLog('Função `updateGraphData` chamada!', 'step');
            
            // Armazena o dado recebido exatamente como ele é.
            latestReceivedData = data; 
            
            // Loga o tipo e uma amostra do conteúdo para o console.
            addLog(`Tipo de 'data' recebido: ${typeof data}.`, 'info');
            let dataSample;
            if (typeof data === 'object' && data !== null) {
                try {
                    dataSample = JSON.stringify(data).substring(0, 200) + '...';
                } catch (e) {
                    dataSample = String(data).substring(0, 200) + '...';
                }
            } else {
                dataSample = String(data).substring(0, 200) + '...';
            }
            addLog(`Amostra do conteúdo: ${dataSample}`, 'info');

            // Dispara uma tentativa imediata de exibição após receber o dado.
            // O intervalo de 3 segundos continuará a verificar e exibir caso a
            // primeira tentativa falhe por questões de DOM ou outras.
            displayReceivedData(); 
        };

        // Quando o documento HTML estiver completamente carregado
        document.addEventListener('DOMContentLoaded', () => {
            addLog('Evento DOMContentLoaded disparado. HTML e DOM estão prontos.', 'success');
            // Marca o DOM como pronto.
            // Isso evita tentativas de manipular elementos que ainda não existem.
            // A flag é usada dentro de `displayReceivedData`.
            // Removed domReady = true; as it's not strictly necessary with this polling approach, 
            // the checks inside displayReceivedData are sufficient.
            
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
