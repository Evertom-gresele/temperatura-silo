<!DOCTYPE 11 html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Silo Termometria</title>
    <style>
        body {
            margin: 0;
            overflow: hidden; /* Evita barras de rolagem desnecessárias */
            font-family: Arial, sans-serif;
            background-color: #f0f0f0; /* Cor de fundo suave */
        }
        #graph-container {
            width: 100vw; /* Largura total da viewport */
            height: 100vh; /* Altura total da viewport */
            display: flex;
            justify-content: center;
            align-items: center;
        }
        .node circle {
            fill: #fff;
            stroke: steelblue;
            stroke-width: 3px;
        }
        .node text {
            font: 10px sans-serif;
            fill: #333;
        }
        .link {
            fill: none;
            stroke: #ccc;
            stroke-width: 2px;
        }
    </style>
    <script src="https://d3js.org/d3.v7.min.js"></script> 
</head>
<body>
    <div id="graph-container">
        <svg id="siloGraph"></svg>
    </div>

    <script>
        let latestReceivedData = null; // Armazena os últimos dados recebidos
        let graphInitialized = false; // Flag para saber se o gráfico foi inicializado

        // --- Funções de Processamento de Dados e Desenho do Gráfico ---

        // Função para manipular o JSON recebido do FlutterFlow
        function handleWebViewJson(jsonData) {
            console.log('handleWebViewJson: Dados recebidos no WebView:', jsonData);

            try {
                // Tenta fazer o parse do JSON. Se já for um objeto JS, não fará nada.
                const parsedData = typeof jsonData === 'string' ? JSON.parse(jsonData) : jsonData;

                if (parsedData && Array.isArray(parsedData.silos)) {
                    latestReceivedData = parsedData; // Armazena os dados mais recentes
                    console.log("handleWebViewJson: Dados JSON válidos. Iniciando atualização do gráfico.");
                    updateGraphData(parsedData); // Chama a função para atualizar o gráfico
                } else {
                    console.warn("handleWebViewJson: JSON não contém uma estrutura 'silos' válida ou não pôde ser parseado.");
                }
            } catch (e) {
                console.error("handleWebViewJson: Erro ao fazer parse do JSON ou processar dados:", e, jsonData);
            }
        }

        // Função para desenhar ou atualizar o gráfico
        function updateGraphData(data) {
            console.log('updateGraphData: Atualizando gráfico com dados:', data);

            const svg = d3.select("#siloGraph");
            const width = window.innerWidth;
            const height = window.innerHeight;

            // Limpa o SVG existente se o gráfico já foi inicializado
            if (graphInitialized) {
                svg.selectAll("*").remove();
            } else {
                svg.attr("width", width)
                   .attr("height", height);
                graphInitialized = true; // Marca como inicializado
            }

            // A lógica de desenho do seu gráfico D3.js aqui
            // Este é um exemplo simplificado de um gráfico de pontos
            const nodes = data.silos.map(silo => ({
                id: silo.siloId,
                x: Math.random() * width, // Posição aleatória para demonstração
                y: Math.random() * height,
                temperatura: silo.temperatura // Exemplo de dado a ser exibido
            }));

            const g = svg.append("g");

            g.selectAll("circle")
                .data(nodes)
                .enter().append("circle")
                .attr("r", 20)
                .attr("cx", d => d.x)
                .attr("cy", d => d.y)
                .attr("fill", "orange")
                .attr("stroke", "red")
                .attr("stroke-width", 2);

            g.selectAll("text")
                .data(nodes)
                .enter().append("text")
                .attr("x", d => d.x)
                .attr("y", d => d.y + 5)
                .attr("text-anchor", "middle")
                .text(d => `${d.siloId}: ${d.temperatura}°C`)
                .style("font-size", "12px")
                .style("fill", "black");

            // Adapte esta seção para a sua lógica de gráfico real (nós, links, layout, etc.)
            console.log("updateGraphData: Gráfico atualizado.");
        }

        // --- Funções de Comunicação com FlutterFlow (Bridge) ---

        // Função para solicitar dados do FlutterFlow através do bridge
        async function requestDataFromFlutter() {
            console.log('requestDataFromFlutter: Solicitando dados ao FlutterFlow...');
            // Verifica se o bridge do FlutterFlow está disponível (necessário para comunicação)
            if (window.flutter_inappwebview) {
                try {
                    // Chama a Ação Personalizada 'getSiloJsonDataFromFlutter' do FlutterFlow
                    // O nome 'getSiloJsonDataFromFlutter' DEVE corresponder ao nome que você deu à Ação Personalizada no FlutterFlow
                    const jsonData = await window.flutter_inappwebview.callHandler('getSiloJsonDataFromFlutter');
                    
                    if (jsonData) {
                        console.log('requestDataFromFlutter: Dados recebidos do FlutterFlow:', jsonData);
                        // Chama a função para atualizar o gráfico com os dados recebidos
                        handleWebViewJson(jsonData); 
                    } else {
                        console.warn('requestDataFromFlutter: Nenhum dado recebido do FlutterFlow.');
                    }
                } catch (error) {
                    console.error('requestDataFromFlutter: Erro ao solicitar dados do FlutterFlow:', error);
                }
            } else {
                console.warn('requestDataFromFlutter: flutter_inappwebview não está disponível. Não é possível solicitar dados.');
            }
        }

        // --- Inicialização ---

        // Chama a função para solicitar dados do FlutterFlow assim que o DOM estiver completamente carregado
        // Isso garante que o WebView tentará obter dados assim que for carregado
        document.addEventListener('DOMContentLoaded', () => {
             console.log('DOMContentLoaded: WebView carregado. Solicitando dados iniciais.');
             requestDataFromFlutter();
        });

        // Opcional: Você pode querer que o WebView puxe os dados periodicamente,
        // mas é melhor que o FlutterFlow (se pudesse) notificasse o WebView de uma atualização.
        // Já que não podemos, essa pode ser uma alternativa, mas use com cautela para não sobrecarregar.
        // Exemplo: a cada 5 segundos
        // setInterval(requestDataFromFlutter, 5000); 

    </script>
</body>
</html>
