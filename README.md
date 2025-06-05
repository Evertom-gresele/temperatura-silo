<!DOCTYPE 33 html>
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
        const targetString = "JSON final enviado para o WebView:";
        const jsonOutputElement = document.getElementById("jsonOutput");
        const statusElement = document.getElementById("status");
        let jsonFound = false;
        let retryCount = 0;
        const maxRetries = 20; // Limite máximo de tentativas

        // Array para armazenar logs do console
        const consoleLogs = [];

        // Sobrescreve o console.log para capturar as mensagens
        const originalConsoleLog = console.log;
        console.log = function(...args) {
            originalConsoleLog.apply(console, args);
            const logString = args.map(arg => {
                if (typeof arg === 'object') {
                    // Se o argumento for um objeto, converte para string JSON
                    return JSON.stringify(arg);
                }
                return String(arg);
            }).join(" ");
            consoleLogs.push(logString);
            
            // Verifica imediatamente se o novo log contém o JSON
            checkForJson(logString);
        };

        function checkForJson(log) {
            if (jsonFound) return;
            
            if (log.includes(targetString)) {
                try {
                    let jsonData;
                    
                    // Usa uma regex mais robusta para encontrar o objeto JSON após o marcador
                    // Isso lida com casos onde o JSON é logado como um objeto separado ou como parte de uma string
                    const match = log.match(new RegExp(targetString + '\\s*(\\{[\\s\\S]*\\})'));
                    
                    if (match && match[1]) {
                        jsonData = JSON.parse(match[1]);
                    }
                    
                    if (jsonData) {
                        jsonOutputElement.textContent = JSON.stringify(jsonData, null, 2);
                        statusElement.textContent = "JSON encontrado e processado com sucesso!";
                        statusElement.classList.add("success");
                        jsonFound = true;
                        originalConsoleLog("JSON encontrado e exibido com sucesso!");
                    }
                } catch (e) {
                    originalConsoleLog("Erro ao parsear JSON:", e);
                    statusElement.textContent = "Erro ao processar JSON: " + e.message;
                }
            }
        }

        function searchAndDisplayJson() {
            if (jsonFound || retryCount >= maxRetries) return;
            
            retryCount++;
            statusElement.textContent = `Buscando JSON... (tentativa ${retryCount}/${maxRetries})`;
            
            // Verifica todos os logs armazenados
            for (const log of consoleLogs) {
                checkForJson(log);
                if (jsonFound) break;
            }

            if (!jsonFound) {
                originalConsoleLog(`JSON não encontrado ainda. Tentativa ${retryCount}/${maxRetries}. Tentando novamente em 3 segundos...`);
                if (retryCount < maxRetries) {
                    setTimeout(searchAndDisplayJson, 3000);
                } else {
                    statusElement.textContent = "Limite de tentativas atingido. JSON não encontrado.";
                }
            }
        }

        // Inicia a busca após um pequeno atraso para garantir que logs iniciais sejam capturados
        // Certifique-se de que os elementos 'jsonOutput' e 'status' existam no DOM antes de executar este script.
        window.onload = () => {
            setTimeout(searchAndDisplayJson, 1000); // Espera 1 segundo antes da primeira busca
        };
    </script>
</body>
</html>

