<!DOCTYPE html>
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
    <p>Aguardando dados... (Verifique o console para logs detalhados)</p>

    <div id="data-display">
        </div>

    <script>
        // Função para exibir dados na tela
        function displayData(data) {
            const dataDisplayDiv = document.getElementById('data-display');
            dataDisplayDiv.innerHTML = ''; // Limpa o conteúdo anterior

            if (!data) {
                dataDisplayDiv.innerHTML = '<p>Nenhum dado recebido ou dado inválido.</p>';
                return;
            }

            // Exibir a string JSON bruta
            const rawDataSection = document.createElement('div');
            rawDataSection.className = 'data-section';
            rawDataSection.innerHTML = '<h2>Dados Brutos (String JSON)</h2><pre>' + escapeHtml(JSON.stringify(data, null, 2)) + '</pre>';
            dataDisplayDiv.appendChild(rawDataSection);

            // Tentar exibir dados parseados de forma mais amigável
            try {
                const parsed = JSON.parse(data);

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
            
            // O FlutterFlow geralmente envia a string JSON já encadeada com aspas
            // Remova as aspas extras se existirem (ex: '"{\"key\":\"value\"}"' se torna '{"key":"value"}')
            let cleanedDataString = data;
            if (typeof data === 'string' && data.startsWith('"') && data.endsWith('"')) {
                cleanedDataString = data.substring(1, data.length - 1);
            }
            
            // Exibir os dados na tela
            displayData(cleanedDataString);
            
            // Opcional: Tentar parsear para confirmar se é JSON válido para depuração no console
            try {
                const parsed = JSON.parse(cleanedDataString);
                console.log("updateGraphData: Dados JSON parseados:", parsed);
            } catch (e) {
                console.error("updateGraphData: Erro ao parsear JSON:", e, "String recebida:", cleanedDataString);
            }
        };

        // Adiciona um listener para quando a página é completamente carregada,
        // para garantir que 'data-display' esteja disponível.
        document.addEventListener('DOMContentLoaded', (event) => {
            console.log("Documento HTML carregado.");
            // Você pode adicionar uma chamada de teste aqui se quiser ver um valor padrão
            // window.updateGraphData(JSON.stringify({ test: "Dados padrão do HTML" }));
        });

    </script>
</body>
</html>
