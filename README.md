<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Silo Termometria</title>
    <script src="https://d3js.org/d3.v7.min.js"></script> 
    <style>
        /* Seu CSS para o gráfico */
        body { margin: 0; overflow: hidden; }
        #graph-container { width: 100vw; height: 100vh; display: flex; justify-content: center; align-items: center; }
    </style>
</head>
<body>
    <div id="graph-container">
        <svg id="siloGraph"></svg>
    </div>

    <script>
        // Função para receber e processar o JSON do FlutterFlow
        function handleWebViewJson(jsonData) {
            console.log('handleWebViewJson: Dados recebidos no WebView:', jsonData);
            try {
                const parsedData = typeof jsonData === 'string' ? JSON.parse(jsonData) : jsonData;
                if (parsedData && Array.isArray(parsedData.silos)) {
                    updateGraphData(parsedData); // Chama a função para atualizar o gráfico
                } else {
                    console.warn("handleWebViewJson: JSON não contém uma estrutura 'silos' válida ou não pôde ser parseado.");
                }
            } catch (e) {
                console.error("handleWebViewJson: Erro ao fazer parse do JSON ou processar dados:", e, jsonData);
            }
        }

        // Sua função para desenhar/atualizar o gráfico com D3.js
        function updateGraphData(data) {
            console.log('updateGraphData: Atualizando gráfico com dados:', data);
            const svg = d3.select("#siloGraph");
            const width = window.innerWidth;
            const height = window.innerHeight;

            svg.selectAll("*").remove(); // Limpa o SVG antes de redesenhar
            svg.attr("width", width).attr("height", height);

            // Lógica D3.js para criar/atualizar o gráfico com 'data.silos'
            // Exemplo básico:
            const nodes = data.silos.map(silo => ({
                id: silo.siloId,
                temperatura: silo.temperatura,
                x: Math.random() * width,
                y: Math.random() * height
            }));

            svg.selectAll("circle")
               .data(nodes)
               .enter().append("circle")
               .attr("r", 20)
               .attr("cx", d => d.x)
               .attr("cy", d => d.y)
               .attr("fill", "blue");

            svg.selectAll("text")
               .data(nodes)
               .enter().append("text")
               .attr("x", d => d.x)
               .attr("y", d => d.y + 5)
               .attr("text-anchor", "middle")
               .text(d => `${d.id}: ${d.temperatura}°C`)
               .style("font-size", "10px")
               .style("fill", "white");
        }

        console.log('index.html script carregado.');
    </script>
</body>
</html>
