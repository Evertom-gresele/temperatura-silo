<!DOCTYPE 1 html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Dados do Silo - Teste de Recebimento</title>
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
            overflow-x: auto; /* Para lidar com JSONs muito longos */
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
    </style>
</head>
<body>
    <h1>Dados Recebidos do FlutterFlow</h1>
    <p id="status-message">Aguardando dados... (Verifique o console para logs detalhados)</p>

    <div id="data-display">
        </div>

    <script>
        let isDomReady = false;
        let pendingData = null; // Para armazenar dados se eles chegarem antes do DOM estar pronto

        // Função para exibir dados na tela
        function displayData(data) {
            const dataDisplayDiv = document.getElementById('data-display');
            const statusMessage = document.getElementById('status-message');

            if (!dataDisplayDiv || !statusMessage) {
                console.warn("displayData: Elementos DOM não encontrados. Dados serão exibidos assim que o DOM estiver pronto.");
                pendingData = data; // Armazena os dados para exibir depois
                return; // Sai da função, espera o DOM ficar pronto
            }

            // Se chegamos aqui, o DOM está pronto e os elementos existem
            if (statusMessage) {
                statusMessage.textContent = "Dados recebidos e exibidos:";
            }
            dataDisplayDiv.innerHTML = ''; // Limpa o conteúdo anterior

            if (!data) {
                dataDisplayDiv.innerHTML = '<p>Nenhum dado recebido ou dado inválido.</p>';
                return;
            }

            // Exibir a string JSON bruta
            const rawDataSection = document.createElement('div');
            rawDataSection.className = 'data-section';
            rawDataSection.innerHTML = '<h2>Dados Brutos (String JSON)</h2><pre>' + escapeHtml(data) + '</pre>'; // Use 'data' diretamente aqui, pois já foi limpado
            dataDisplayDiv.appendChild(rawDataSection);

            // Tentar exibir dados parseados de forma mais amigável
            try {
                const parsed = JSON.parse(data); // Tenta parsear a string limpa

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
                errorSection.innerHTML = '<h2>Erro ao Parsear JSON</h2><p>String recebida não é um JSON válido.</p><pre>' + escapeHtml(data) + '</pre><p>Erro: ' + escapeHtml(e.message) + '</p>';
                dataDisplayDiv.appendChild(errorSection);
            }
        }

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

        // Esta é a função que o FlutterFlow irá chamar
        window.updateGraphData = function(data) {
            console.log("updateGraphData: Função chamada com dados brutos do FlutterFlow:", data);
            
            let cleanedDataString = data;
            // Se o FlutterFlow envia a string JSON já encadeada com aspas (ex: '"{\"key\":\"value\"}"'), remove as aspas extras
            if (typeof data === 'string' && data.startsWith('"') && data.endsWith('"')) {
                cleanedDataString = data.substring(1, data.length - 1);
                console.log("updateGraphData: String JSON limpa (removidas aspas extras):", cleanedDataString);
            }
            
            // Se o DOM já está pronto, exibe os dados diretamente
            if (isDomReady) {
                displayData(cleanedDataString);
            } else {
                // Caso contrário, armazena os dados para processar quando o DOM estiver pronto
                pendingData = cleanedDataString;
                console.log("updateGraphData: DOM ainda não pronto, dados armazenados para exibição posterior.");
            }
            
            // Opcional: Tentar parsear para confirmar se é JSON válido para depuração no console
            try {
                const parsed = JSON.parse(cleanedDataString);
                console.log("updateGraphData: Dados JSON parseados:", parsed);
            } catch (e) {
                console.error("updateGraphData: Erro ao parsear JSON no console (não impede exibição se a string for válida):", e, "String recebida:", cleanedDataString);
            }
        };

        // Adiciona um listener para quando a página é completamente carregada
        document.addEventListener('DOMContentLoaded', (event) => {
            console.log("Documento HTML carregado. DOM pronto.");
            isDomReady = true; // Define a flag para indicar que o DOM está pronto

            // Se houver dados pendentes que chegaram antes do DOM estar pronto, exiba-os agora
            if (pendingData) {
                console.log("DOMContentLoaded: Exibindo dados pendentes.");
                displayData(pendingData);
                pendingData = null; // Limpa os dados pendentes
            }
        });

    </script>
</body>
</html>
