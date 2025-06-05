<!DOCTYPE 4 html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Dados do Silo - Teste Final</title>
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
    </style>
</head>
<body>
    <h1>Dados Recebidos do FlutterFlow</h1>
    <p id="status-message">Aguardando dados... (Verifique o console para logs detalhados)</p>

    <div id="data-display">
        </div>

    <script>
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
            // Remove aspas extras se existirem
            if (typeof data === 'string' && data.startsWith('"') && data.endsWith('"')) {
                cleanedDataString = data.substring(1, data.length - 1);
                console.log("updateGraphData: String JSON limpa (removidas aspas extras):", cleanedDataString);
            }
            
            // Tentar parsear para confirmar se é JSON válido para depuração no console
            try {
                const parsed = JSON.parse(cleanedDataString);
                console.log("updateGraphData: Dados JSON parseados para console:", parsed);
            } catch (e) {
                console.error("updateGraphData: Erro ao parsear JSON para console:", e, "String recebida:", cleanedDataString);
                // Se não conseguir parsear, não há sentido em tentar exibir
                return; 
            }

            // Tenta obter os elementos DOM *imediatamente*
            const dataDisplayDiv = document.getElementById('data-display');
            const statusMessage = document.getElementById('status-message');

            if (!dataDisplayDiv || !statusMessage) {
                console.error("updateGraphData: Elementos DOM (data-display ou status-message) NÃO encontrados ao tentar exibir. O WebView pode estar recarregando ou o DOM ainda não está pronto para manipulação.");
                // Retorna. Os dados foram recebidos, mas não puderam ser exibidos no momento.
                return; 
            }

            // Se chegamos aqui, o DOM está pronto e os elementos existem
            console.log("updateGraphData: Elementos DOM encontrados, procedendo com a exibição.");
            statusMessage.textContent = "Dados recebidos e exibidos:";
            dataDisplayDiv.innerHTML = ''; // Limpa o conteúdo anterior

            // Exibir a string JSON bruta
            const rawDataSection = document.createElement('div');
            rawDataSection.className = 'data-section';
            rawDataSection.innerHTML = '<h2>Dados Brutos (String JSON)</h2><pre>' + escapeHtml(cleanedDataString) + '</pre>';
            dataDisplayDiv.appendChild(rawDataSection);

            // Exibir dados parseados de forma mais amigável (usando 'parsed' do try-catch acima)
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
        };

        // Não há listener DOMContentLoaded ou lógica de pendingData aqui,
        // pois a ideia é que a função tente exibir os dados imediatamente
        // e logue se os elementos não estiverem prontos.
        console.log("Script index3.html carregado.");

    </script>
</body>
</html>
