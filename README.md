<!DOCTYPE 10 html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Dados do Silo - Sincronização Flexível</title>
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
            color: #d63384; /* Um pouco mais visível para o status */
        }
        .error-message {
            color: #dc3545;
            font-weight: bold;
        }
    </style>
</head>
<body>
    <h1>Dados Recebidos do FlutterFlow</h1>
    <p id="status-message">Aguardando dados... (Verifique o console para logs detalhados)</p>

    <div id="data-display">
        </div>

    <script>
        let latestReceivedData = null; // Armazena sempre o último dado recebido bruto (string JSON)
        let renderAttemptInterval = null; // Para controlar o intervalo de tentativas de renderização

        // Função auxiliar para escapar HTML (segurança básica)
        function escapeHtml(text) {
            const map = {
                '&': '&amp;',
                '<': '&lt;',
                '>': '&gt;',
                '"': '&quot;',
                "'": '&#039;'
            };
            return text.replace(/[&<>"']/g, function(m) { return map[m]; });
        }

        // Função para tentar exibir os dados no DOM
        function attemptToDisplayData() {
            const dataDisplayDiv = document.getElementById('data-display');
            const statusMessage = document.getElementById('status-message');

            // 1. Verificar se os elementos DOM essenciais estão disponíveis
            if (!dataDisplayDiv || !statusMessage) {
                console.warn("attemptToDisplayData: Elementos DOM (data-display ou status-message) não encontrados. Re-tentando em 1 segundo...");
                // Não inicia o intervalo aqui, a função `updateGraphData` ou `DOMContentLoaded` o fará.
                return; // Sai e espera a próxima tentativa via intervalo.
            }

            // Se chegamos aqui, o DOM está pronto. Parar as re-tentativas.
            if (renderAttemptInterval) {
                clearInterval(renderAttemptInterval);
                renderAttemptInterval = null;
                console.log("attemptToDisplayData: Elementos DOM encontrados e intervalo de re-tentativa parado.");
            }
            
            // 2. Verificar se há dados para exibir
            if (!latestReceivedData) {
                console.log("attemptToDisplayData: DOM pronto, mas nenhum dado recebido ainda. Exibindo mensagem de aguardo.");
                statusMessage.textContent = "Aguardando dados...";
                dataDisplayDiv.innerHTML = ''; // Limpa qualquer conteúdo anterior se não houver dados.
                return; 
            }

            // Se chegamos aqui, o DOM está pronto e há dados. Proceder com a exibição.
            console.log("attemptToDisplayData: DOM pronto e dados disponíveis. Exibindo dados.");
            statusMessage.textContent = "Dados recebidos e exibidos:";
            dataDisplayDiv.innerHTML = ''; // Limpa o conteúdo anterior

            // Exibir a string JSON bruta
            const rawDataSection = document.createElement('div');
            rawDataSection.className = 'data-section';
            rawDataSection.innerHTML = '<h2>Dados Brutos (String JSON)</h2><pre>' + escapeHtml(latestReceivedData) + '</pre>';
            dataDisplayDiv.appendChild(rawDataSection);

            // Tentar exibir dados parseados de forma mais amigável
            try {
                const parsed = JSON.parse(latestReceivedData);

                // Exibir distribuicaoCabos
                const distCabosSection = document.createElement('div');
                distCabosSection.className = 'data-section';
                distCabosSection.innerHTML = '<h2>distribuicaoCabos</h2><pre>' + escapeHtml(JSON.stringify(parsed.distribuicaoCabos, null, 2)) + '</pre>';
                dataDisplayDiv.appendChild(distCabosSection);

                // Exibir alturaCabos
                const alturaCabosSection = document.createElement('div');
                alturaCabosSection.className = 'data-section';
                alturaCabosSection.innerHTML = '<h2>alturaCabos</h2><pre>' + escapeHtml(JSON.stringify(parsed.alturaCabos, null, 2)) + '</pre>';
                dataDisplayDiv.appendChild(alturaCabosSection);

                // Exibir leiturasTemperatura
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

        // Esta é a função que o FlutterFlow irá chamar
        window.updateGraphData = function(data) {
            console.log("updateGraphData: Função chamada com dados brutos do FlutterFlow:", data);
            
            let cleanedDataString = data;
            // Se o FlutterFlow envia a string JSON já encadeada com aspas (ex: '"{\"key\":\"value\"}"'), remove as aspas extras
            if (typeof data === 'string' && data.startsWith('"') && data.endsWith('"')) {
                cleanedDataString = data.substring(1, data.length - 1);
                console.log("updateGraphData: String JSON limpa (removidas aspas extras):", cleanedDataString);
            }
            
            latestReceivedData = cleanedDataString; // Armazena sempre o dado mais recente
            console.log("updateGraphData: Dados armazenados em latestReceivedData.");

            // Inicia a tentativa de renderização imediatamente, e se não conseguir,
            // um intervalo de 1 segundo será iniciado/mantido pela chamada inicial
            // de DOMContentLoaded ou por chamadas anteriores.
            attemptToDisplayData();

            // Opcional: Tentar parsear para confirmar se é JSON válido para depuração no console
            try {
                const parsed = JSON.parse(cleanedDataString);
                console.log("updateGraphData: Dados JSON parseados para console:", parsed);
            } catch (e) {
                console.error("updateGraphData: Erro ao parsear JSON para console:", e, "String recebida:", cleanedDataString);
            }
        };

        // Adiciona um listener para quando a página é completamente carregada
        document.addEventListener('DOMContentLoaded', (event) => {
            console.log("Documento HTML carregado. DOM pronto. Iniciando/Reiniciando tentativas de exibição.");
            
            // Inicia o intervalo de re-tentativa se ainda não estiver ativo
            if (!renderAttemptInterval) {
                renderAttemptInterval = setInterval(attemptToDisplayData, 1000); // Tenta a cada 1 segundo
            }
            
            // Faz uma tentativa imediata após o DOM estar pronto
            attemptToDisplayData(); 
        });

        // Limpa o intervalo se o iframe for descarregado
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
