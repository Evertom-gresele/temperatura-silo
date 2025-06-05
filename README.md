<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Dados do Silo - Teste Final de DOM</title>
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

    <script>
        let latestReceivedData = null; 
        let renderAttemptInterval = null; 
        let domAccessAttempts = 0; // Contador para depuração

        function escapeHtml(text) {
            const map = { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#039;' };
            return text.replace(/[&<>"']/g, function(m) { return map[m]; });
        }

        function attemptToDisplayData() {
            domAccessAttempts++;
            const dataDisplayDiv = document.getElementById('data-display');
            const statusMessage = document.getElementById('status-message');
            const runtimeContentDiv = document.getElementById('runtime-content'); // Novo elemento de teste

            if (!dataDisplayDiv || !statusMessage || !runtimeContentDiv) {
                console.warn(`attemptToDisplayData: Tentativa ${domAccessAttempts}. Elementos DOM não encontrados. Re-tentando em 1 segundo...`);
                // Adiciona um elemento simples ao corpo para testar o DOM mais primitivamente
                if (document.body && !document.getElementById('body-test-div')) {
                    const testDiv = document.createElement('div');
                    testDiv.id = 'body-test-div';
                    testDiv.textContent = `Teste de body.appendChild: Tentativa ${domAccessAttempts}. DOM elements not found yet.`;
                    testDiv.style.border = '1px solid blue';
                    testDiv.style.margin = '5px';
                    document.body.appendChild(testDiv);
                    console.log("attemptToDisplayData: Adicionado div de teste ao body.");
                } else if (document.getElementById('body-test-div')) {
                    document.getElementById('body-test-div').textContent = `Teste de body.appendChild: Tentativa ${domAccessAttempts}. DOM elements not found yet.`;
                }
                return;
            }

            // Se chegamos aqui, o DOM está pronto. Parar as re-tentativas.
            if (renderAttemptInterval) {
                clearInterval(renderAttemptInterval);
                renderAttemptInterval = null;
                console.log("attemptToDisplayData: Elementos DOM encontrados e intervalo de re-tentativa parado.");
            }
            
            // Adiciona uma mensagem de sucesso no elemento de teste runtime-content
            runtimeContentDiv.innerHTML = `<p style="color: green; font-weight: bold;">Sucesso! Elementos DOM encontrados após ${domAccessAttempts} tentativas.</p>`;

            // ... (Restante da lógica de exibição de dados, que é a mesma do código anterior) ...
            if (!latestReceivedData) {
                console.log("attemptToDisplayData: DOM pronto, mas nenhum dado recebido ainda. Exibindo mensagem de aguardo.");
                statusMessage.textContent = "Aguardando dados...";
                dataDisplayDiv.innerHTML = ''; 
                return; 
            }

            console.log("attemptToDisplayData: DOM pronto e dados disponíveis. Exibindo dados.");
            statusMessage.textContent = "Dados recebidos e exibidos:";
            dataDisplayDiv.innerHTML = ''; 

            const rawDataSection = document.createElement('div');
            rawDataSection.className = 'data-section';
            rawDataSection.innerHTML = '<h2>Dados Brutos (String JSON)</h2><pre>' + escapeHtml(latestReceivedData) + '</pre>';
            dataDisplayDiv.appendChild(rawDataSection);

            try {
                const parsed = JSON.parse(latestReceivedData);

                const distCabosSection = document.createElement('div');
                distCabosSection.className = 'data-section';
                distCabosSection.innerHTML = '<h2>distribuicaoCabos</h2><pre>' + escapeHtml(JSON.stringify(parsed.distribuicaoCabos, null, 2)) + '</pre>';
                dataDisplayDiv.appendChild(distCabosSection);

                const alturaCabosSection = document.createElement('div');
                alturaCabosSection.className = 'data-section';
                alturaCabosSection.innerHTML = '<h2>alturaCabos</h2><pre>' + escapeHtml(JSON.stringify(parsed.alturaCabos, null, 2)) + '</pre>';
                dataDisplayDiv.appendChild(alturaCabosSection);

                const leiturasTempSection = document.createElement('div');
                leiturasTempSection.className = 'data-section';
                leiturasTempSection.innerHTML = '<h2>leiturasTemperatura</h2><pre>' + escapeHtml(JSON.stringify(parsed.leiturasTemperatura, null, 2)) + '</pre>';
                dataDisplayDiv.appendChild(leiturasTempSection);

            } catch (e) {
                const errorSection = document.createElement('div');
                errorSection.className = 'data-section';
                errorSection.innerHTML = '<h2 class="error-message">Erro ao Parsear JSON para Exibição Detalhada</h2><p>String recebida não é um JSON válido.</p><pre class="error-message">' + escapeHtml(latestReceivedData) + '</pre><p class="error-message">Erro: ' + escapeHtml(e.message) + '</p>';
                dataDisplayDiv.appendChild(errorSection);
            }
        }

        window.updateGraphData = function(data) {
            console.log("updateGraphData: Função chamada com dados brutos do FlutterFlow:", data);
            
            let cleanedDataString = data;
            if (typeof data === 'string' && data.startsWith('"') && data.endsWith('"')) {
                cleanedDataString = data.substring(1, data.length - 1);
                console.log("updateGraphData: String JSON limpa (removidas aspas extras):", cleanedDataString);
            }
            
            latestReceivedData = cleanedDataString; 
            console.log("updateGraphData: Dados armazenados em latestReceivedData. Tentando exibir...");

            // Tenta exibir imediatamente. Se não conseguir, o setInterval se encarregará.
            attemptToDisplayData();

            try {
                const parsed = JSON.parse(cleanedDataString);
                console.log("updateGraphData: Dados JSON parseados para console:", parsed);
            } catch (e) {
                console.error("updateGraphData: Erro ao parsear JSON para console:", e, "String recebida:", cleanedDataString);
            }
        };

        document.addEventListener('DOMContentLoaded', (event) => {
            console.log("Documento HTML carregado. DOM pronto. Iniciando/Reiniciando tentativas de exibição.");
            
            // Inicia o intervalo de re-tentativa se ainda não estiver ativo
            if (!renderAttemptInterval) {
                renderAttemptInterval = setInterval(attemptToDisplayData, 1000); 
            }
            
            // Faz uma tentativa imediata após o DOM estar pronto
            attemptToDisplayData(); 
        });

        window.addEventListener('beforeunload', () => {
            if (renderAttemptInterval) {
                clearInterval(renderAttemptInterval);
                renderAttemptInterval = null;
                console.log("beforeunload: Intervalo de re-tentativa limpo.");
            }
        });

        console.log("Script index3.html carregado.");

    </script>
</body>
</html>
