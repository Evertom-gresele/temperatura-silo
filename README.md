<!DOCTYPE 6 html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Dados do Silo - Debug Variável</title>
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
        // Variáveis globais
        let latestReceivedData = null; // Armazena os dados crus do FlutterFlow como string
        let renderAttemptInterval = null; // ID do setInterval
        let domAccessAttempts = 0; // Contador de tentativas de acesso ao DOM
        let logCounter = 0; // Para numerar as etapas no log
        let dataHasBeenProcessed = false; // Flag para indicar se os dados foram processados com sucesso

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
            // Garante que o texto seja uma string antes de tentar o replace
            const stringText = String(text); 
            return stringText.replace(/[&<>"']/g, function(m) { return map[m]; });
        }

        // Função principal para tentar exibir os dados no DOM
        function attemptToDisplayData() {
            domAccessAttempts++;
            addLog(`Executando attemptToDisplayData. Tentativa: ${domAccessAttempts}.`, 'step');

            // 1. Tentar acessar elementos DOM
            const dataDisplayDiv = document.querySelector('#data-display');
            const statusMessage = document.querySelector('#status-message');
            const runtimeContentDiv = document.querySelector('#runtime-content');

            // 2. Logar o conteúdo do body para inspeção (truncado para evitar logs enormes)
            if (document.body) {
                const bodyContent = document.body.innerHTML;
                addLog(`Conteúdo do document.body na tentativa ${domAccessAttempts}: ${bodyContent.substring(0, Math.min(bodyContent.length, 500))}... (truncado para 500 chars)`, 'info');
            } else {
                addLog(`document.body não acessível na tentativa ${domAccessAttempts}.`, 'warn');
            }

            // 3. Verificar se todos os elementos necessários foram encontrados no DOM
            if (!dataDisplayDiv || !statusMessage || !runtimeContentDiv) {
                addLog(`Elementos DOM (data-display, status-message, ou runtime-content) NÃO encontrados. Re-tentando em 1 segundo...`, 'warn');
                
                // Teste de appendChild simples para confirmar que o body é manipulável
                if (document.body) {
                    let bodyTestDiv = document.querySelector('#body-test-div');
                    if (!bodyTestDiv) { 
                        bodyTestDiv = document.createElement('div');
                        bodyTestDiv.id = 'body-test-div';
                        bodyTestDiv.style.border = '1px solid blue';
                        bodyTestDiv.style.margin = '5px';
                        document.body.appendChild(bodyTestDiv);
                        addLog("Adicionado div de teste ao body ('#body-test-div').", 'info');
                    }
                    bodyTestDiv.textContent = `Teste de body.appendChild: Tentativa ${domAccessAttempts}. Elementos DOM ainda não encontrados.`;
                } else {
                    addLog("Não foi possível adicionar div de teste: document.body não está disponível.", 'error');
                }

                // Loga o estado do latestReceivedData AQUI para debug
                const displayDataInfo = latestReceivedData === null ? 'null' : (typeof latestReceivedData === 'string' ? `string (length: ${latestReceivedData.length})` : typeof latestReceivedData);
                addLog(`Estado de latestReceivedData: ${displayDataInfo}. Conteúdo: ${latestReceivedData ? String(latestReceivedData).substring(0, Math.min(String(latestReceivedData).length, 100)) + '...' : 'N/A'}`, 'info');
                
                // Manter a mensagem "Aguardando dados..." na tela se o DOM não estiver pronto
                if (statusMessage) { // Verifica se statusMessage foi encontrado para não dar erro
                    statusMessage.textContent = "Aguardando dados... (DOM não pronto)";
                }
                return; // Sai da função, o setInterval irá chamar novamente
            }

            // Se chegamos aqui, o DOM está pronto. Parar as re-tentativas de DOM.
            if (renderAttemptInterval) {
                clearInterval(renderAttemptInterval);
                renderAttemptInterval = null;
                addLog(`Sucesso! Elementos DOM encontrados após ${domAccessAttempts} tentativas. Parando re-tentativas de DOM.`, 'success');
                runtimeContentDiv.innerHTML = `<p style="color: green; font-weight: bold;">Sucesso! Elementos DOM encontrados após ${domAccessAttempts} tentativas.</p>`;
            } else {
                addLog(`Elementos DOM já estavam encontrados.`, 'info');
            }
            
            // Agora, vamos verificar se os dados foram recebidos e exibidos
            if (dataHasBeenProcessed) {
                addLog("Dados já foram processados e exibidos. Nenhuma ação adicional.", 'info');
                return; // Não precisa re-renderizar se já foi feito
            }

            // Lógica para verificar e exibir os dados
            if (latestReceivedData !== null) { // Agora verifica se não é null
                addLog("Dados enviados pelo FlutterFlow ENCONTRADOS na variável `latestReceivedData`!", 'success');
                statusMessage.textContent = "Dados recebidos e exibidos (raw):";
                dataDisplayDiv.innerHTML = ''; // Limpa o conteúdo anterior

                const rawDataSection = document.createElement('div');
                rawDataSection.className = 'data-section';
                // Garante que o conteúdo seja tratado como string para exibição
                rawDataSection.innerHTML = '<h2>Conteúdo Bruto Recebido</h2><pre>' + escapeHtml(String(latestReceivedData)) + '</pre>';
                dataDisplayDiv.appendChild(rawDataSection);
                addLog("Conteúdo bruto adicionado ao DOM. Tipo: " + typeof latestReceivedData, 'success');

                // *** REMOVIDA A TENTATIVA DE PARSE JSON AQUI ***
                // O objetivo é APENAS exibir o que foi recebido.
                // A lógica de parsing e exibição de JSON detalhado
                // pode ser adicionada posteriormente, uma vez que
                // entendamos o formato bruto.

                dataHasBeenProcessed = true; // Marca que os dados foram exibidos
                addLog("Exibição de dados brutos concluída.", 'success');
            } else {
                addLog("Dados enviados pelo FlutterFlow NÃO encontrados em `latestReceivedData`. Aguardando...", 'warn');
                statusMessage.textContent = "Aguardando dados...";
            }
        }

        // Esta é a função que o FlutterFlow irá chamar
        window.updateGraphData = function(data) {
            addLog("Função updateGraphData chamada pelo FlutterFlow.", 'step');
            addLog(`Tipo de 'data' recebido: ${typeof data}. Conteúdo bruto recebido em updateGraphData: ${String(data).substring(0, Math.min(String(data).length, 500))}... (truncado para 500 chars)`, 'info');
            
            // Simplesmente armazena o dado recebido, convertendo-o para string se não for
            latestReceivedData = String(data); 
            addLog(`latestReceivedData ATUALIZADO em updateGraphData. Valor: ${latestReceivedData.substring(0, Math.min(latestReceivedData.length, 100))}...`, 'info');

            // Imediatamente tenta exibir os dados após recebê-los
            attemptToDisplayData();
            
            // Não fazemos mais nenhum JSON.parse ou limpeza aqui.
            // O objetivo é capturar o dado EXATAMENTE como ele chega.
        };

        // Adiciona um listener para quando a página é completamente carregada
        document.addEventListener('DOMContentLoaded', (event) => {
            addLog("Evento DOMContentLoaded disparado. Documento HTML carregado. DOM pronto.", 'success');
            
            // Inicia o intervalo de re-tentativa para DOM e dados a cada 1 segundo
            if (!renderAttemptInterval) {
                renderAttemptInterval = setInterval(attemptToDisplayData, 1000); // Tenta a cada 1 segundo
                addLog("Intervalo de re-tentativa (1s) iniciado para DOM e dados.", 'info');
            } else {
                addLog("Intervalo de re-tentativa já estava ativo.", 'info');
            }
            
            // Faz uma tentativa imediata após o DOM estar pronto
            attemptToDisplayData(); 
        });

        // Limpa o intervalo se o iframe for descarregado
        window.addEventListener('beforeunload', () => {
            addLog("Evento beforeunload disparado. Limpando intervalo de re-tentativa.", 'info');
            if (renderAttemptInterval) {
                clearInterval(renderAttemptInterval);
                renderAttemptInterval = null;
            }
        });

        addLog("Script index3.html carregado.", 'success');

    </script>
</body>
</html>
