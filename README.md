<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Receptor Simples WebView</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 20px;
            background-color: #e0f7fa; /* Cor de fundo leve para visualização */
            color: #004d40;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 90vh; /* Ocupa a maior parte da tela */
        }
        h1 {
            color: #00796b;
            text-align: center;
            margin-bottom: 20px;
        }
        pre {
            background-color: #ffffff;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 4px 8px rgba(0,0,0,0.1);
            white-space: pre-wrap; /* Permite quebras de linha longas */
            word-wrap: break-word; /* Quebra palavras longas */
            max-width: 90%; /* Limita a largura para melhor visualização */
            overflow-x: auto;
            border: 1px solid #b2dfdb;
            font-size: 0.9em;
        }
        #status-message {
            font-weight: bold;
            color: #d84315; /* Cor de alerta */
            margin-bottom: 15px;
            padding: 8px;
            background-color: #ffe0b2; /* Fundo de alerta */
            border-radius: 5px;
        }
        .success {
            color: #388e3c; /* Cor de sucesso */
            background-color: #c8e6c9; /* Fundo de sucesso */
        }
    </style>
</head>
<body>
    <h1>Dados Recebidos pelo WebView</h1>
    <div id="status-message">Aguardando dados...</div>
    <pre id="received-data-output">Nenhum dado recebido ainda.</pre>

    <script>
        // Esta é a ÚNICA função JavaScript que o FlutterFlow precisa chamar.
        // Ela vai receber os dados e exibi-los na tela e no console do NAVEGADOR.
        window.handleWebViewJson = function(data) {
            console.log('handleWebViewJson: Dados recebidos. Tipo:', typeof data, 'Conteúdo:', data);

            const statusMessage = document.getElementById('status-message');
            const outputElement = document.getElementById('received-data-output');

            if (statusMessage && outputElement) {
                let contentToDisplay;
                // Tenta formatar como JSON bonito se for um objeto ou uma string JSON
                if (typeof data === 'object' && data !== null) {
                    try {
                        contentToDisplay = JSON.stringify(data, null, 2);
                        console.log('handleWebViewJson: Dados eram objeto, stringificado para JSON.');
                    } catch (e) {
                        contentToDisplay = String(data) + '\n(Erro ao formatar objeto JSON: ' + e.message + ')';
                        console.error('handleWebViewJson: Erro ao stringify o objeto:', e);
                    }
                } else {
                    // Se já for uma string, tenta parsear e formatar como JSON
                    try {
                        const parsedData = JSON.parse(data);
                        contentToDisplay = JSON.stringify(parsedData, null, 2);
                        console.log('handleWebViewJson: Dados eram string, tentando parsear e formatar como JSON.');
                    } catch (e) {
                        // Se não for objeto nem string JSON válida, exibe como string bruta
                        contentToDisplay = String(data) + '\n(Não é um objeto nem JSON válido)';
                        console.warn('handleWebViewJson: Dados não são JSON válido nem objeto, exibindo como string bruta.');
                    }
                }
                
                outputElement.textContent = contentToDisplay;
                statusMessage.textContent = 'Dados Recebidos com Sucesso!';
                statusMessage.className = 'success'; // Adiciona classe para mudar cor
            } else {
                console.error('handleWebViewJson: Elementos DOM (status-message ou received-data-output) não encontrados. Verifique o HTML.');
            }
        };

        console.log('HTML do WebView carregado. Aguardando chamada handleWebViewJson...');
    </script>
</body>
</html>
