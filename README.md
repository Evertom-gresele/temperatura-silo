<!DOCTYPE 2 html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>JSON do WebView</title>
    <style>
        body {
            font-family: monospace;
            margin: 20px;
            background-color: #f5f5f5;
        }
        h1 {
            color: #333;
            border-bottom: 1px solid #ccc;
            padding-bottom: 10px;
        }
        pre {
            background-color: #fff;
            border: 1px solid #ddd;
            border-radius: 5px;
            padding: 15px;
            white-space: pre-wrap;
            overflow-x: auto;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        .status {
            color: #666;
            font-style: italic;
            margin-bottom: 10px;
        }
        .success {
            color: green;
        }
    </style>
</head>
<body>
    <h1>JSON Recebido do WebView</h1>
    <div id="status" class="status">Aguardando JSON...</div>
    <pre id="jsonOutput">Aguardando dados...</pre>

    <script>
        // Função para receber o JSON diretamente do FlutterFlow via evaluateJavascript (como window.updateGraphData ou window.handleWebViewJson)
        window.handleWebViewJson = function(jsonData) {
            console.log('handleWebViewJson: Dados recebidos diretamente:', jsonData); // Loga o que foi recebido!
            const jsonOutputElement = document.getElementById("jsonOutput");
            const statusElement = document.getElementById("status");
            
            if (jsonOutputElement && statusElement) {
                let displayContent;
                if (typeof jsonData === 'object' && jsonData !== null) {
                    try {
                        displayContent = JSON.stringify(jsonData, null, 2);
                    } catch (e) {
                        displayContent = String(jsonData) + '\n(Erro ao formatar JSON: ' + e.message + ')';
                        console.error('handleWebViewJson: Erro ao stringify JSON:', e);
                    }
                } else {
                    // Se o dado não é um objeto (ex: já é uma string JSON), tenta parsear e formatar
                    try {
                        const parsedData = JSON.parse(jsonData);
                        displayContent = JSON.stringify(parsedData, null, 2);
                    } catch (e) {
                        displayContent = String(jsonData) + '\n(Dado não é JSON válido ou objeto)';
                        console.warn('handleWebViewJson: Dados não são um objeto ou JSON válido, exibindo como string.', e);
                    }
                }

                jsonOutputElement.textContent = displayContent;
                statusElement.textContent = "JSON recebido diretamente da WebView!";
                statusElement.classList.add("success");
            } else {
                console.warn('handleWebViewJson: Elementos DOM não encontrados para atualização.');
            }
        };

        // Função para buscar o JSON no console (interceptando console.log)
        // ATENÇÃO: Esta abordagem é mais frágil e normalmente não é recomendada para produção.
        // A função 'print' do Flutter envia para o console do Flutter/Dart, não necessariamente para o console JavaScript do WebView.
        // A função abaixo TENTA interceptar, mas pode não funcionar para logs vindos do Dart/Flutter diretamente.
        (function() {
            const targetString = "JSON final enviado para o WebView:";
            let jsonFoundInConsole = false; // Renomeado para evitar conflito e clareza
            let retryCount = 0;
            const maxRetries = 20;
            const consoleLogs = []; // Array para armazenar logs capturados

            // Sobrescreve o console.log para capturar as mensagens
            const originalConsoleLog = console.log;
            console.log = function(...args) {
                originalConsoleLog.apply(console, args); // Garante que o log original ainda funcione
                
                if (jsonFoundInConsole) return; // Se já encontrou, não precisa mais processar logs

                // Converte todos os argumentos para uma única string para processamento
                const logString = args.map(arg => {
                    if (typeof arg === 'object') {
                        try {
                            return JSON.stringify(arg);
                        } catch (e) {
                            return String(arg); // Fallback para objetos complexos
                        }
                    }
                    return String(arg);
                }).join(" ");
                
                consoleLogs.push(logString); // Armazena o log

                // Processa o log imediatamente ao ser capturado
                if (logString.includes(targetString)) {
                    try {
                        // Regex para extrair o JSON: busca a string alvo seguida por espaços e um objeto JSON {...}
                        const match = logString.match(new RegExp(targetString + '\\s*(\\{[\\s\\S]*\\})'));
                        if (match && match[1]) {
                            const jsonData = JSON.parse(match[1]); // Tenta fazer o parse do JSON extraído
                            const jsonOutputElement = document.getElementById("jsonOutput");
                            const statusElement = document.getElementById("status");
                            
                            if (jsonOutputElement && statusElement) {
                                jsonOutputElement.textContent = JSON.stringify(jsonData, null, 2);
                                statusElement.textContent = "JSON encontrado e processado via interceptação do console!";
                                statusElement.classList.add("success");
                                jsonFoundInConsole = true; // Marca como encontrado
                                originalConsoleLog("JSON encontrado e exibido com sucesso via console interceptado!");
                            }
                        }
                    } catch (e) {
                        originalConsoleLog("Erro ao processar JSON da interceptação do console:", e);
                    }
                }
            };

            // Função que tenta buscar o JSON nos logs capturados ou no console (recursivamente)
            function searchAndDisplayJsonFromCapturedLogs() {
                if (jsonFoundInConsole || retryCount >= maxRetries) {
                    if (retryCount >= maxRetries && !jsonFoundInConsole) {
                        const statusElement = document.getElementById("status");
                        if (statusElement) {
                            statusElement.textContent = "Limite de tentativas de busca no console atingido. JSON não encontrado por essa via.";
                            console.error('searchAndDisplayJsonFromCapturedLogs: Limite de tentativas atingido. JSON não encontrado.');
                        }
                    }
                    return;
                }
                
                retryCount++;
                const statusElement = document.getElementById("status");
                if (statusElement) {
                    statusElement.textContent = `Buscando JSON no console... (tentativa ${retryCount}/${maxRetries})`;
                }
                
                // Reprocessa todos os logs armazenados (principalmente para logs que ocorreram antes do DOMContentLoaded)
                for (const log of consoleLogs) {
                    if (log.includes(targetString) && !jsonFoundInConsole) {
                        // Re-chamar a lógica de interceptação para garantir que o JSON seja processado
                        // (o console.log sobrescrito já faz a maior parte do trabalho, mas garante que o DOM seja atualizado).
                        // Esta parte é mais um fallback, o principal é o console.log interceptado.
                        // Basicamente, força a reavaliação de logs antigos.
                    }
                }

                if (!jsonFoundInConsole) {
                    originalConsoleLog(`JSON não encontrado ainda via interceptação. Tentativa ${retryCount}/${maxRetries}. Tentando novamente em 1 segundo...`);
                    if (retryCount < maxRetries) {
                        setTimeout(searchAndDisplayJsonFromCapturedLogs, 1000);
                    }
                }
            }

            // Inicia a busca imediatamente após o script ser executado
            // (pode ser antes de alguns logs aparecerem, por isso o setTimeout e o array consoleLogs).
            searchAndDisplayJsonFromCapturedLogs();
        })();
    </script>
</body>
</html>
