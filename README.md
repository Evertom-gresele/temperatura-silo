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
        // Dados de exemplo (simulando a entrada do FlutterFlow)
        const sampleData = {
            "distribuicaoCabos": [
                {
                    "tipoUnidade": 1,
                    "linha_1": 1,
                    "linha_2": 3
                }
            ],
            "alturaCabos": {
                "cabo_1": 16,
                "cabo_2": 14,
                "cabo_3": 14,
                "cabo_4": 14
            },
            "leiturasTemperatura": {
                "cabo_1": [
                    32, 31, 31, 30, 25, 25, 26, 24, 25, 24, 26, 24, 26, 24, 23, 24
                ],
                "cabo_2": [
                    36, 34, 35, 24, 21, 24, 23, 25, 26, 24, 25, 23, 24, 21
                ],
                "cabo_3": [
                    36, 34, 35, 24, 21, 24, 23, 25, 26, 24, 25, 23, 24, 21
                ],
                "cabo_4": [
                    36, 34, 35, 24, 21, 24, 23, 25, 26, 24, 25, 23, 24, 21
                ]
            }
        };

        // ==========================================================
        //  FUNÇÕES AUXILIARES
        // ==========================================================

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
            textGeometry.translate( - textGeometry.boundingBox.max.x / 2, 0, 0 ); // Centraliza o texto horizontalmente

            const textMaterial = new THREE.MeshBasicMaterial({ color: 0x000000 }); // Cor do texto (preto)
            const textMesh = new THREE.Mesh(textGeometry, textMaterial);
            return textMesh;
        }

        // Função para criar e posicionar os cabos com base nos dados
        function createSiloCables(data) {
            // Parâmetros para a distribuição de altura
            const cableHeightUnit = 2;
            const cableWidth = 2;

            const distribuicao = data.distribuicaoCabos[0];
            const alturaCabos = data.alturaCabos;
            const leiturasTemperatura = data.leiturasTemperatura;

            let numLines = 0;
            for (const key in distribuicao) {
                if (key.startsWith('linha_')) {
                    numLines++;
                }
            }

            // Cache para fontes (para evitar recarregar a cada texto)
            const fontLoader = new THREE.FontLoader();
            // Carrega a fonte uma vez e depois cria o texto para todos os segmentos
            // Usaremos uma fonte padrão Three.js disponível em CDN
            fontLoader.load('https://unpkg.com/three@0.128.0/examples/fonts/helvetiker_regular.typeface.json', function (font) {
                let loadedFont = font; // A fonte carregada é passada para esta variável
                let caboCounter = 1;

                for (let i = 1; i <= numLines; i++) {
                    const currentLineKey = `linha_${i}`;
                    const numCabosInLine = distribuicao[currentLineKey];

                    const currentRadius = (i - 1) * 20;

                    for (let j = 0; j < numCabosInLine; j++) {
                        const angle = (j / numCabosInLine) * Math.PI * 2;

                        const x = currentRadius * Math.cos(angle);
                        const z = currentRadius * Math.sin(angle);

                        const caboId = `cabo_${caboCounter}`;
                        const alturaCabo = alturaCabos[caboId];
                        const leiturasCabo = leiturasTemperatura[caboId];

                        if (alturaCabo && leiturasCabo) {
                            const geometry = new THREE.BoxGeometry(cableWidth, cableHeightUnit, cableWidth);

                            for (let k = 0; k < alturaCabo; k++) {
                                const temp = leiturasCabo[k];
                                const color = getColorForTemperature(temp);

                                const material = new THREE.MeshBasicMaterial({ color: color });

                                const segment = new THREE.Mesh(geometry, material);
                                segment.position.set(x, (k * cableHeightUnit) + (cableHeightUnit / 2), z);
                                segment.name = 'cableSegment';
                                scene.add(segment);

                                // Adiciona o texto da temperatura ao lado do segmento
                                if (loadedFont) {
                                    const textMesh = createTemperatureText(temp.toString(), loadedFont);
                                    // Posição do texto: ligeiramente ao lado do segmento
                                    textMesh.position.set(x + cableWidth * 0.8, (k * cableHeightUnit) + (cableHeightUnit / 2), z);
                                    textMesh.name = 'temperatureText';
                                    scene.add(textMesh);
                                }
                            }
                        } else {
                            console.warn(`Dados incompletos para o ${caboId}.`);
                        }
                        caboCounter++;
                    }
                }
            }); // Fim do fontLoader.load
        }


        // ==========================================================
        //  CONFIGURAÇÃO DA CENA THREE.JS
        // ==========================================================

        let scene, camera, renderer, controls;

        function init() {
            // Cena
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0xf0f0f0);

            // Câmera
            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.set(0, 50, 100);

            // Renderizador
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            document.body.appendChild(renderer.domElement);

            // OrbitControls
            controls = new THREE.OrbitControls(camera, renderer.domElement);
            controls.enableDamping = true;
            controls.dampingFactor = 0.05;
            controls.screenSpacePanning = false;
            controls.minDistance = 10;
            controls.maxDistance = 500;
            controls.maxPolarAngle = Math.PI / 2;
            controls.target.set(0, 20, 0);
            controls.update();

            // Luzes
            const ambientLight = new THREE.AmbientLight(0x404040);
            scene.add(ambientLight);
            const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
            directionalLight.position.set(5, 10, 7.5);
            scene.add(directionalLight);

            // Eixos (para ajudar na visualização inicial)
            const axesHelper = new THREE.AxesHelper(100);
            scene.add(axesHelper);

            // Calcular o raio do silo dinamicamente com base nas linhas
            const distribuicaoData = sampleData.distribuicaoCabos[0];
            let numLines = 0;
            for (const key in distribuicaoData) {
                if (key.startsWith('linha_')) {
                    numLines++;
                }
            }
            const siloRadius = ((numLines - 1) * 20) + 20;

            // Cilindro de exemplo para representar o silo
            const siloGeometry = new THREE.CylinderGeometry(siloRadius, siloRadius, 40, 32);
            const siloMaterial = new THREE.MeshStandardMaterial({
                color: 0xc0c0c0,
                transparent: true,
                opacity: 0.2,
                side: THREE.DoubleSide
            });
            const silo = new THREE.Mesh(siloGeometry, siloMaterial);
            silo.position.y = 20;
            scene.add(silo);

            // Chamada da função para criar os cabos
            createSiloCables(sampleData);

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

        // Função para receber dados do FlutterFlow (será implementada mais tarde)
        function updateGraphData(data) {
        // Função para receber dados do FlutterFlow e atualizar o gráfico
        function updateGraphData(data) {
            console.log("Dados recebidos do FlutterFlow para atualização:", data);

            // 1. Remover todos os objetos de cabos e textos existentes da cena
            const objectsToRemove = [];
            scene.children.forEach(object => {
                if (object.name === 'cableSegment' || object.name === 'temperatureText') {
                    objectsToRemove.push(object);
                }
            });

            objectsToRemove.forEach(object => {
                // Libera a geometria e o material da memória para evitar vazamento
                if (object.geometry) object.geometry.dispose();
                if (object.material) object.material.dispose();
                scene.remove(object);
            });

            // 2. Chamar createSiloCables com os NOVOS dados
            // NOTA: Ajuste o sampleData para o 'data' real recebido do FlutterFlow
            createSiloCables(data);

            // 3. (Opcional, mas recomendado) Ajustar o tamanho do silo se a distribuição mudar
            const distribuicaoData = data.distribuicaoCabos[0];
            let numLines = 0;
            for (const key in distribuicaoData) {
                if (key.startsWith('linha_')) {
                    numLines++;
                }
            }
            const newSiloRadius = ((numLines - 1) * 20) + 20;

            // Encontre o objeto do silo na cena e atualize sua geometria
            let siloMesh = null;
            scene.children.forEach(object => {
                // Assumindo que o silo é o único CylinderGeometry grande ou você pode dar um nome a ele
                if (object.geometry instanceof THREE.CylinderGeometry && object.position.y === 20) {
                    siloMesh = object;
                }
            });

            if (siloMesh && siloMesh.geometry.parameters.radiusTop !== newSiloRadius) {
                // Remove a geometria antiga e cria uma nova
                siloMesh.geometry.dispose();
                siloMesh.geometry = new THREE.CylinderGeometry(newSiloRadius, newSiloRadius, 40, 32);
                console.log(`Raio do silo atualizado para: ${newSiloRadius}`);
            }
        }        }
    </script>
</body>
</html>
