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
        let axesHelper; // Variável para o AxesHelper

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

            // Eixos (agora com tamanho dinâmico do silo)
            // Para que os eixos não "passem" do silo, o tamanho do AxesHelper deve ser o RAIO para X e Z.
            // Para o Y, o tamanho deve ser a altura total do cilindro (maxCableHeight).
            // No entanto, AxesHelper desenha todos os eixos com o mesmo comprimento.
            // A melhor abordagem é fazer o tamanho do AxesHelper igual ao raio do silo (para X e Z)
            // e ajustá-lo na posição para o Y.
            const axesLength = siloRadius; // Comprimento dos eixos X e Z para ir do centro até a borda do silo
            axesHelper = new THREE.AxesHelper(axesLength);
            // Ajusta a posição do AxesHelper para que a ponta do eixo Y chegue até a altura máxima dos cabos
            // A origem do AxesHelper é o centro. Se o eixo Y for 'axesLength', ele vai de 0 a axesLength.
            // Queremos que ele se estenda do "chão" (Y=0) até maxCableHeight.
            // Como AxesHelper desenha 0 a `size` para cada eixo, para o Y, se queremos que vá até `maxCableHeight`,
            // o `size` deve ser `maxCableHeight` e a origem `0`.
            // Mas X e Z precisam ir do centro até o raio.
            // Uma solução é ter um AxesHelper para a origem e outro para a altura, ou usar um valor que abranja
            // as dimensões mais importantes.
            // Para que as linhas "sigam" o silo, vamos fazer o length do AxesHelper igual ao diâmetro do silo (para X e Z)
            // e posicioná-lo no Y para que a linha Y vá do chão até o topo do cilindro.
            const visualAxesLength = Math.max(siloRadius * 2, maxCableHeight); // Usa o maior entre diâmetro e altura do cilindro para a escala
            axesHelper = new THREE.AxesHelper(visualAxesLength);
            // Como o AxesHelper desenha do ponto de origem até o valor do size,
            // vamos posicionar sua origem para que a linha Y (verde) vá do 0 até o maxCableHeight.
            // Se o tamanho do AxesHelper é `visualAxesLength`, e o eixo Y vai de 0 a `visualAxesLength`,
            // para que a linha verde termine em `maxCableHeight`, o size deve ser `maxCableHeight`.
            // Mas isso faria X e Z terem `maxCableHeight` de comprimento.
            // A melhor forma de "limitar" visualmente é usar o raio para X e Z.
            // E o Y é a altura. AxesHelper não é flexível nesse sentido.

            // Vamos tentar uma abordagem onde o tamanho do AxesHelper é o raio do silo para X e Z,
            // e para o Y, a linha do eixo vai do chão até o topo do cilindro.
            // Three.js AxesHelper desenha todos os eixos com o mesmo comprimento.
            // Se queremos que ele se limite ao silo, o comprimento máximo deve ser o diâmetro do silo (2*siloRadius).
            // A altura do silo é `maxCableHeight`.
            // Para que o eixo Y "cubra" a altura do cilindro, o tamanho deve ser `maxCableHeight`.
            // Mas X e Z seriam muito grandes se `maxCableHeight` for maior que o diâmetro.
            // Se `size` é o raio, X e Z vão de -raio a +raio.
            // O eixo Y vai de 0 a raio.

            // O jeito mais direto com AxesHelper é que ele seja o tamanho do diâmetro horizontal
            // e o posicionamos no chão. A linha azul e vermelha vão do -raio ao +raio.
            // A linha verde vai do 0 ao raio.
            axesHelper = new THREE.AxesHelper(siloRadius);
            axesHelper.position.y = 0; // A base dos eixos está no chão, no centro do silo.
            scene.add(axesHelper);

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
                siloBaseCone.geometry.dispose(); // Necessário para alterar o raio se ele mudar
                siloBaseCone.geometry = new THREE.ConeGeometry(newSiloRadius, coneHeight, 32);
                siloBaseCone.position.y = -coneHeight / 2;
                siloBaseCone.rotation.x = Math.PI; // Garante que continue invertido
            }

            // Atualiza a posição e raio do cone superior
            if (siloTopCone) {
                siloTopCone.geometry.dispose(); // Necessário para alterar o raio se ele mudar
                siloTopCone.geometry = new THREE.ConeGeometry(newSiloRadius, coneHeight, 32);
                siloTopCone.position.y = newMaxCableHeight + (coneHeight / 2);
            }

            // Atualiza o AxesHelper para o novo tamanho do silo
            if (axesHelper) {
                scene.remove(axesHelper); // Remove o antigo
                // O tamanho do AxesHelper agora é o raio do silo
                axesHelper = new THREE.AxesHelper(newSiloRadius);
                axesHelper.position.y = 0; // Mantém a base no chão
                scene.add(axesHelper); // Adiciona o novo
            }

            // 4. Chamar createSiloCables com os NOVOS dados para desenhar os cabos e textos
            createSiloCables(data);
        }
    </script>
</body>
</html>
