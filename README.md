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
            // textGeometry.translate( - textGeometry.boundingBox.max.x / 2, 0, 0 ); // Removido ou comentado

            const textMaterial = new THREE.MeshBasicMaterial({ color: 0x000000 }); // Cor do texto (preto)
            const textMesh = new THREE.Mesh(textGeometry, textMaterial);
            return textMesh;
        }

        // Função para criar e posicionar os cabos com base nos dados
        function createSiloCables(data) {
            // Parâmetros para a distribuição de altura
            const cableHeightUnit = 4; // Dobrando o espaçamento entre os pontos
            const cableWidth = 2;
            const textHorizontalOffset = 1.5; // Ajuste este valor para mover o texto para o lado (um pouco mais que metade da largura do cabo)

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
                                    // Posição do texto: ajustando para ficar ao lado do cabo
                                    // Adicionamos a metade da largura do cabo + um offset
                                    textMesh.position.set(x + cableWidth / 2 + textHorizontalOffset, (k * cableHeightUnit) + (cableHeightUnit / 2), z);
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
        let siloMesh, siloBaseCone, siloTopCone; // Variáveis para armazenar os meshes do silo

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
            controls.target.set(0, 40, 0);
            controls.update();

            // Luzes
            const ambientLight = new THREE.AmbientLight(0x404040);
            scene.add(ambientLight);
            const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
            directionalLight.position.set(5, 20, 15);
            scene.add(directionalLight);

            // Eixos (para ajudar na visualização inicial)
            const axesHelper = new THREE.AxesHelper(200);
            scene.add(axesHelper);

            // ==========================================================
            //  Criação do Silo e Funis (Atualizado)
            // ==========================================================
            const siloMaterial = new THREE.MeshStandardMaterial({
                color: 0xc0c0c0,
                transparent: true,
                opacity: 0.2,
                side: THREE.DoubleSide
            });

            // Determinar a altura máxima dos cabos para a altura do cilindro
            let maxCableHeight = 0;
            const cableHeightUnit = 4; // Deve ser o mesmo valor definido em createSiloCables
            for (const caboId in sampleData.alturaCabos) {
                const currentCableTotalHeight = sampleData.alturaCabos[caboId] * cableHeightUnit;
                if (currentCableTotalHeight > maxCableHeight) {
                    maxCableHeight = currentCableTotalHeight;
                }
            }

            // Raio do silo dinamicamente com base nas linhas
            const distribuicaoData = sampleData.distribuicaoCabos[0];
            let numLines = 0;
            for (const key in distribuicaoData) {
                if (key.startsWith('linha_')) {
                    numLines++;
                }
            }
            const siloRadius = ((numLines - 1) * 20) + 20;
            const coneHeight = 15; // Altura dos cones superior e inferior
            const coneBaseRadius = siloRadius; // Raio da base dos cones

            // Cilindro principal (corpo do silo)
            const siloGeometry = new THREE.CylinderGeometry(siloRadius, siloRadius, maxCableHeight, 32);
            siloMesh = new THREE.Mesh(siloGeometry, siloMaterial);
            siloMesh.position.y = maxCableHeight / 2; // Centraliza o cilindro em sua própria altura
            siloMesh.name = 'siloCylinder';
            scene.add(siloMesh);

            // Cone inferior (funil da base)
            const baseConeGeometry = new THREE.ConeGeometry(coneBaseRadius, coneHeight, 32);
            siloBaseCone = new THREE.Mesh(baseConeGeometry, siloMaterial);
            siloBaseCone.position.y = -coneHeight / 2; // Posiciona abaixo do "chão" dos cabos
            siloBaseCone.rotation.x = Math.PI; // Inverte o cone para baixo
            siloBaseCone.name = 'siloBaseCone';
            scene.add(siloBaseCone);

            // Cone superior (topo do silo)
            const topConeGeometry = new THREE.ConeGeometry(coneBaseRadius, coneHeight, 32);
            siloTopCone = new THREE.Mesh(topConeGeometry, siloMaterial);
            siloTopCone.position.y = maxCableHeight + (coneHeight / 2); // Posiciona acima do cilindro
            siloTopCone.name = 'siloTopCone';
            scene.add(siloTopCone);

            // ==========================================================
            // Fim da criação do Silo e Funis
            // ==========================================================

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
            console.log("Dados recebidos do FlutterFlow:", data);

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

            // 2. Determinar a nova altura máxima dos cabos para a altura do cilindro
            let newMaxCableHeight = 0;
            const cableHeightUnit = 4; // Deve ser o mesmo valor definido em createSiloCables
            for (const caboId in data.alturaCabos) {
                const currentCableTotalHeight = data.alturaCabos[caboId] * cableHeightUnit;
                if (currentCableTotalHeight > newMaxCableHeight) {
                    newMaxCableHeight = currentCableTotalHeight;
                }
            }

            // 3. Atualizar a altura do cilindro principal e a posição dos cones
            const distribuicaoData = data.distribuicaoCabos[0];
            let numLines = 0;
            for (const key in distribuicaoData) {
                if (key.startsWith('linha_')) {
                    numLines++;
                }
            }
            const newSiloRadius = ((numLines - 1) * 20) + 20;
            const coneHeight = 15;

            // Atualiza o cilindro principal
            if (siloMesh) {
                siloMesh.geometry.dispose(); // Libera a geometria antiga
                siloMesh.geometry = new THREE.CylinderGeometry(newSiloRadius, newSiloRadius, newMaxCableHeight, 32);
                siloMesh.position.y = newMaxCableHeight / 2;
                console.log(`Cilindro atualizado: Raio=${newSiloRadius}, Altura=${newMaxCableHeight}`);
            }

            // Atualiza a posição do cone inferior
            if (siloBaseCone) {
                // Se o raio mudar, pode ser necessário recriar o cone, mas apenas a posição Y é suficiente aqui
                siloBaseCone.position.y = -coneHeight / 2;
            }

            // Atualiza a posição do cone superior
            if (siloTopCone) {
                // Se o raio mudar, pode ser necessário recriar o cone, mas apenas a posição Y é suficiente aqui
                siloTopCone.position.y = newMaxCableHeight + (coneHeight / 2);
            }

            // 4. Chamar createSiloCables com os NOVOS dados para desenhar os cabos e textos
            createSiloCables(data);
        }
    </script>
</body>
</html>
