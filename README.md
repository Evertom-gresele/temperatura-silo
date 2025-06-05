<!DOCTYPE2 html>
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
    <script>
        const targetString = "JSON final enviado para o WebView:";
        let jsonOutputElement; // Será inicializado após o DOM carregar
        let statusElement;     // Será inicializado após o DOM carregar
        let jsonFound = false;
        let retryCount = 0;
        const maxRetries = 20; // Limite máximo de tentativas

        // Array para armazenar logs do console
        const consoleLogs = [];

        // Sobrescreve o console.log para capturar as mensagens
        const originalConsoleLog = console.log;
        console.log = function(...args) {
            originalConsoleLog.apply(console, args);
            
            // Converte todos os argumentos para uma única string para processamento robusto
            const logString = args.map(arg => {
                if (typeof arg === 'object') {
                    return JSON.stringify(arg);
                }
                return String(arg);
            }).join(" ");
            
            consoleLogs.push(logString);
            processConsoleLogs(); // Processa logs imediatamente
        };

        function processConsoleLogs() {
            if (jsonFound) return; // Se o JSON já foi encontrado, não faz nada
            
            for (const log of consoleLogs) {
                if (log.includes(targetString)) {
                    try {
                        // Regex para capturar o JSON que está em formato de string
                        // Procura por '{' e '}' que delimitam o objeto JSON
                        const match = log.match(new RegExp(targetString + '\\s*(\\{[\\s\\S]*\\})'));
                        
                        if (match && match[1]) {
                            const jsonData = JSON.parse(match[1]);
                            // Garante que os elementos do DOM existam antes de tentar acessá-los
                            if (jsonOutputElement && statusElement) {
                                jsonOutputElement.textContent = JSON.stringify(jsonData, null, 2);
                                statusElement.textContent = "JSON encontrado e processado com sucesso!";
                                statusElement.classList.add("success");
                                jsonFound = true;
                                originalConsoleLog("JSON encontrado e exibido com sucesso!");
                                // Limpa os logs para evitar reprocessamento desnecessário
                                consoleLogs.length = 0;
                                return; // Sai da função assim que encontrar
                            }
                        }
                    } catch (e) {
                        originalConsoleLog("Erro ao parsear JSON de log:", e);
                        if (statusElement) {
                            statusElement.textContent = "Erro ao processar JSON: " + e.message;
                        }
                    }
                }
            }
        }

        function searchAndDisplayJson() {
            // Inicializa os elementos do DOM aqui, pois o script está no head
            if (!jsonOutputElement) {
                jsonOutputElement = document.getElementById("jsonOutput");
            }
            if (!statusElement) {
                statusElement = document.getElementById("status");
            }

            if (jsonFound || retryCount >= maxRetries) return;
            
            retryCount++;
            if (statusElement) {
                statusElement.textContent = `Buscando JSON... (tentativa ${retryCount}/${maxRetries})`;
            }
            
            // Tenta encontrar o JSON nos logs já armazenados (para logs que ocorreram antes da inicialização completa)
            processConsoleLogs();

            if (!jsonFound) {
                originalConsoleLog(`JSON não encontrado ainda. Tentativa ${retryCount}/${maxRetries}. Tentando novamente em 3 segundos...`);
                if (retryCount < maxRetries) {
                    setTimeout(searchAndDisplayJson, 3000);
                } else {
                    if (statusElement) {
                        statusElement.textContent = "Limite de tentativas atingido. JSON não encontrado.";
                    }
                }
            }
        }

        // Inicia a busca após o DOM ser completamente carregado
        document.addEventListener('DOMContentLoaded', () => {
            searchAndDisplayJson();
        });

    </script>
</head>
<body>
    <h1>JSON Recebido do WebView</h1>
    <div id="status" class="status">Aguardando JSON...</div>
    <pre id="jsonOutput">Aguardando dados...</pre>
</body>
</html>

