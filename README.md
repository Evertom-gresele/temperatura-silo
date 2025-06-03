<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Gráfico de Temperaturas do Silo</title>
    <style>
        body { margin: 0; overflow: hidden; }
        canvas { display: block; }
    </style>
</head>
<body>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script src="https://unpkg.com/three@0.128.0/examples/js/controls/OrbitControls.js"></script>
    <script src="https://unpkg.com/three@0.128.0/examples/js/loaders/FontLoader.js"></script>
    <script>
        // Dados de exemplo REMOVIDOS - O FlutterFlow enviará os dados.

        // ==========================================================
        //  VARIÁVEIS GLOBAIS E FUNÇÕES AUXILIARES
        // ==========================================================

        let scene, camera, renderer, controls;
        let siloMesh, siloBaseCone, siloTopCone;
        let axesHelper;
        let loadedFont; // Variável para armazenar a fonte carregada

        // Mapa de cores para interpolação
        const colorMap = [
            { temp: 10, color: new THREE.Color(0x0000FF) }, // Azul (para temperaturas <= 15)
            { temp: 15, color: new THREE.Color(0x0000FF) }, // Azul
            { temp: 19, color: new THREE.Color(0x008000) }, // Verde
            { temp: 25, color: new THREE.Color(0xFFFF00) }, // Amarelo
            { temp: 30, color: new THREE.Color(0xFF0000) }, // Vermelho
            { temp: 35, color: new THREE.Color(0x8B0000) }, // Vermelho Escuro (para temperaturas >= 35)
            { temp: 40, color: new THREE.Color(0x8B0000) }  // Vermelho Escuro (extensão para cima)
        ];

        // Função para mapear temperatura para cor com gradiente suave
        function getColorForTemperature(temp) {
            if (temp <= colorMap[0].temp) {
                return colorMap[0].color;
            }
            if (temp >= colorMap[colorMap.length - 1].temp) {
                return colorMap[colorMap.length - 1].color;
            }

            for (let i = 0; i < colorMap.length - 1; i++) {
                const stop1 = colorMap[i];
                const stop2 = colorMap[i + 1];

                if (temp >= stop1.temp && temp < stop2.temp) {
                    const factor = (temp - stop1.temp) / (stop2.temp - stop1.temp);
                    const interpolatedColor = new THREE.Color();
                    interpolatedColor.copy(stop1.color).lerp(stop2.color, factor);
                    return interpolatedColor;
                }
            }
            return new THREE.Color(0x000000); // Fallback
        }

        // Função para criar o Mesh de texto 3D
        function createTemperatureText(text, font) {
            const textGeometry = new THREE.TextGeometry(text, {
                font: font,
                size: 1.5, // Tamanho do texto
                height: 0.1, // Profundidade do texto 3D
                curveSegments: 12,
                bevelEnabled: false
            });
            textGeometry.computeBoundingBox();

            const textMaterial = new THREE.MeshBasicMaterial({ color: 0x000000 }); // Cor do texto (preto)
            const textMesh = new THREE.Mesh(textGeometry, textMaterial);
            return textMesh;
        }

        // Função para criar e posicionar os cabos com base nos dados
        function createSiloCables(data) {
            const cableHeightUnit = 4;
            const cableWidth = 2;
            const textHorizontalOffset = 1.5;

            const distribuicao = data.distribuicaoCabos; // Agora é um objeto direto
            const alturaCabos = data.alturaCabos;
            const leiturasTemperatura = data.leiturasTemperatura;

            // Certifica-se de que a fonte está carregada antes de criar textos
            if (!loadedFont) {
                console.warn("Fonte ainda não carregada. Tentando novamente em 100ms.");
                setTimeout(() => createSiloCables(data), 100);
                return;
            }

            let caboCounter = 1;

            // Percorre as linhas de 1 a 5 (ou até onde houver dados em distribuicao)
            for (let i = 1; i <= 5; i++) { // Iterar até a linha 5, conforme sua descrição
                const numCabosInLine = distribuicao[`linha_${i}`];

                if (numCabosInLine === undefined || numCabosInLine === 0 || numCabosInLine === "") {
                    continue; // Pula se não houver cabos nesta linha
                }

                const currentRadius = (i - 1) * 20; // Raio para a linha atual (0 para linha_1, 20 para linha_2, etc.)

                for (let j = 0; j < numCabosInLine; j++) {
                    const angle = (j / numCabosInLine) * Math.PI * 2;

                    const x = currentRadius * Math.cos(angle);
                    const z = currentRadius * Math.sin(angle);

                    const caboId = `cabo_${caboCounter}`;
                    const alturaCabo = alturaCabos[caboId];
                    const leiturasCabo = leiturasTemperatura[caboId];

                    if (alturaCabo && leiturasCabo && leiturasCabo.length > 0) {
                        const geometry = new THREE.BoxGeometry(cableWidth, cableHeightUnit, cableWidth);

                        for (let k = 0; k < alturaCabo; k++) {
                            // Verifica se a leitura para este ponto existe
                            if (k < leiturasCabo.length) {
                                const temp = leiturasCabo[k];
                                const color = getColorForTemperature(temp);

                                const material = new THREE.MeshBasicMaterial({ color: color });

                                const segment = new THREE.Mesh(geometry, material);
                                segment.position.set(x, (k * cableHeightUnit) + (cableHeightUnit / 2), z);
                                segment.name = 'cableSegment';
                                scene.add(segment);

                                const textMesh = createTemperatureText(temp.toString(), loadedFont);
                                textMesh.position.set(x + cableWidth / 2 + textHorizontalOffset, (k * cableHeightUnit) + (cableHeightUnit / 2), z);
                                textMesh.name = 'temperatureText';
                                scene.add(textMesh);
                            } else {
                                console.warn(`Leitura de temperatura faltando para ${caboId} no ponto ${k + 1}.`);
                            }
                        }
                    } else {
                        console.warn(`Dados incompletos ou cabo inexistente para o ${caboId}.`);
                    }
                    caboCounter++;
                }
            }
        }


        // ==========================================================
        //  CONFIGURAÇÃO DA CENA THREE.JS
        // ==========================================================

        function init() {
            // Cena
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0xf0f0f0);

            // Câmera
            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 2000);
            camera.position.set(0, 100, 200);

            // Renderizador
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            document.body.appendChild(renderer.domElement);

            // OrbitControls
            controls = new THREE.OrbitControls(camera, renderer.domElement);
            controls.enableDamping = true;
            controls.dampingFactor = 0.05;
            controls.screenSpacePanning = false;
            controls.minDistance = 20;
            controls.maxDistance = 1000;
            controls.maxPolarAngle = Math.PI / 2;
            controls.target.set(0, 40, 0); // Ajusta o ponto de foco inicial
            controls.update();

            // Luzes
            const ambientLight = new THREE.AmbientLight(0x404040);
            scene.add(ambientLight);
            const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
            directionalLight.position.set(5, 20, 15);
            scene.add(directionalLight);

            // Carrega a fonte uma única vez ao iniciar
            const fontLoader = new THREE.FontLoader();
            fontLoader.load('https://unpkg.com/three@0.128.0/examples/fonts/helvetiker_regular.typeface.json', function (font) {
                loadedFont = font;
                console.log("Fonte carregada.");
                // Se houver dados iniciais para carregar, pode chamar updateGraphData aqui após o carregamento da fonte
                // Por enquanto, não chamaremos, pois o FlutterFlow fará a primeira chamada.
            });

            // Lidar com redimensionamento da janela
            window.addEventListener('resize', onWindowResize, false);
        }

        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        }

        function animate() {
            requestAnimationFrame(animate);
            controls.update();
            renderer.render(scene, camera);
        }

        init();
        animate();

        // Função para receber dados do FlutterFlow e atualizar o gráfico
        window.updateGraphData = function(data) { // Torna a função global para ser acessível pelo FlutterFlow
            console.log("Dados recebidos do FlutterFlow:", data);

            // Validar se os dados essenciais estão presentes
            if (!data || !data.distribuicaoCabos || !data.alturaCabos || !data.leiturasTemperatura) {
                console.error("Dados incompletos ou inválidos recebidos do FlutterFlow.");
                return;
            }

            // 1. Remover todos os objetos de cabos e textos existentes da cena
            const objectsToRemove = [];
            scene.children.forEach(object => {
                if (object.name === 'cableSegment' || object.name === 'temperatureText' || object.name === 'siloCylinder' || object.name === 'siloBaseCone' || object.name === 'siloTopCone' || object.name === 'axesHelper') {
                    objectsToRemove.push(object);
                }
            });

            objectsToRemove.forEach(object => {
                if (object.geometry) object.geometry.dispose();
                if (object.material) object.material.dispose();
                scene.remove(object);
            });

            // Limpa as referências globais dos objetos do silo e eixos para recriá-los
            siloMesh = null;
            siloBaseCone = null;
            siloTopCone = null;
            axesHelper = null;


            // 2. Determinar a nova altura máxima dos cabos e o raio do silo para a altura do cilindro
            let newMaxCableHeight = 0;
            const cableHeightUnit = 4;

            for (const caboId in data.alturaCabos) {
                const currentCableTotalHeight = data.alturaCabos[caboId] * cableHeightUnit;
                if (currentCableTotalHeight > newMaxCableHeight) {
                    newMaxCableHeight = currentCableTotalHeight;
                }
            }

            // Se não houver cabos ou altura definida, garanta uma altura mínima para o silo visual
            if (newMaxCableHeight === 0) {
                newMaxCableHeight = 50; // Altura padrão se não houver cabos
            }

            const distribuicaoData = data.distribuicaoCabos;
            let maxLineIndex = 0;
            for (let i = 1; i <= 5; i++) { // Verifica até a linha 5
                if (distribuicaoData[`linha_${i}`] > 0) {
                    maxLineIndex = i;
                }
            }

            // Calcula o raio do silo baseado na linha mais externa com cabos
            // O raio 0 seria para a linha 1 (cabo central), raio 20 para linha 2, etc.
            const newSiloRadius = (maxLineIndex > 0) ? ((maxLineIndex - 1) * 20) + 20 : 20; // Raio mínimo de 20 se não houver linhas

            const coneHeight = 15;

            // Material do silo (reusado ou recriado)
            const siloMaterial = new THREE.MeshStandardMaterial({
                color: 0xc0c0c0,
                transparent: true,
                opacity: 0.2,
                side: THREE.DoubleSide
            });

            // Atualiza/Cria o cilindro principal
            const siloGeometry = new THREE.CylinderGeometry(newSiloRadius, newSiloRadius, newMaxCableHeight, 32);
            siloMesh = new THREE.Mesh(siloGeometry, siloMaterial);
            siloMesh.position.y = newMaxCableHeight / 2;
            siloMesh.name = 'siloCylinder';
            scene.add(siloMesh);

            // Atualiza/Cria o cone inferior
            const baseConeGeometry = new THREE.ConeGeometry(newSiloRadius, coneHeight, 32);
            siloBaseCone = new THREE.Mesh(baseConeGeometry, siloMaterial);
            siloBaseCone.position.y = -coneHeight / 2;
            siloBaseCone.rotation.x = Math.PI;
            siloBaseCone.name = 'siloBaseCone';
            scene.add(siloBaseCone);

            // Atualiza/Cria o cone superior
            const topConeGeometry = new THREE.ConeGeometry(newSiloRadius, coneHeight, 32);
            siloTopCone = new THREE.Mesh(topConeGeometry, siloMaterial);
            siloTopCone.position.y = newMaxCableHeight + (coneHeight / 2);
            siloTopCone.name = 'siloTopCone';
            scene.add(siloTopCone);

            // Atualiza/Cria o AxesHelper
            // Usa o maior entre o raio do silo e a metade da altura para dimensionar os eixos visuais
            const visualAxesLength = Math.max(newSiloRadius, newMaxCableHeight / 2);
            axesHelper = new THREE.AxesHelper(visualAxesLength);
            axesHelper.position.y = 0; // A base dos eixos está no chão, no centro do silo.
            axesHelper.name = 'axesHelper';
            scene.add(axesHelper);

            // 4. Chamar createSiloCables com os NOVOS dados para desenhar os cabos e textos
            createSiloCables(data);
        };
    </script>
</body>
</html>
