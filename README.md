<!DOCTYPE 5 html>
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
    <script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/controls/OrbitControls.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/loaders/FontLoader.js"></script>
    
    <script>
        // ==========================================================
        //  CONFIGURAÇÃO
        // ==========================================================
        const configuracao = {
            distanciaEntreAneisDeCabos: 20,
            distanciaDoUltimoAnelDaParede: 20,
            distanciaEntrePontosDeLeitura: 6,
            diametroDeExibicaoDaTemperatura: 2,
            tamanhoDaFonteDeTemperatura: 2, 
        };

        // ==========================================================
        //  VARIÁVEIS GLOBAIS E FUNÇÕES AUXILIARES
        // ==========================================================

        let scene, camera, renderer, controls;
        let siloMesh, siloTopCone, siloBasePlate;
        let axesHelper;
        let loadedFont;
        let objectsToRemove = []; // Array para armazenar objetos a serem removidos na atualização

        // Configurações do silo
        const coneHeight = 8; // Altura do cone superior
        const pointSpacing = configuracao.distanciaEntrePontosDeLeitura; // Espaçamento vertical entre pontos de temperatura

        // Mapa de cores para interpolação de temperaturas
        const colorMap = [
            { temp: 10, color: new THREE.Color(0x0000ff) }, // Azul (frio)
            { temp: 20, color: new THREE.Color(0x00ffff) }, // Ciano
            { temp: 30, color: new THREE.Color(0x00ff00) }, // Verde
            { temp: 40, color: new THREE.Color(0xffff00) }, // Amarelo
            { temp: 50, color: new THREE.Color(0xffa500) }, // Laranja
            { temp: 60, color: new THREE.Color(0xff0000) }  // Vermelho (quente)
        ];

        // DADOS DE EXEMPLO - ESTES SERÃO APENAS USADOS NA INICIALIZAÇÃO SE NENHUM DADO FOR ENVIADO
        // O `updateGraphData` irá desenhar com os dados recebidos.
        const defaultInitialData = {
            distribuicaoCabos: {
                "linha_1": 1,
                "linha_2": 3,
                "linha_3": 6,
                "linha_4": 0,
                "linha_5": 0
            },
            alturaCabos: {
                "cabo_1": 16,
                "cabo_2": 14,
                "cabo_3": 14,
                "cabo_4": 14,
                "cabo_5": 0,
                "cabo_6": 0,
                "cabo_7": 0,
                "cabo_8": 0,
                "cabo_9": 0,
                "cabo_10": 0,
                "cabo_11": 0,
                "cabo_12": 0,
            },
            leiturasTemperatura: {
                "cabo_1": [10, 12, 15, 18, 20, 22, 25, 28, 30, 32, 35, 38, 40, 42, 45, 48],
                "cabo_2": [15, 18, 20, 22, 25, 28, 30, 32, 35, 38, 40, 42, 45, 48],
                "cabo_3": [20, 22, 25, 26, 15, 28, 30, 32, 35, 38, 40, 42, 45, 48],
                "cabo_4": [25, 28, 29, 26, 25, 28, 30, 32, 35, 38, 40, 42, 45, 48],
            }
        };

        // Função para interpolar cores baseado na temperatura
        function getColorForTemperature(temp) {
            if (temp <= colorMap[0].temp) return colorMap[0].color;
            if (temp >= colorMap[colorMap.length - 1].temp) return colorMap[colorMap.length - 1].color;

            for (let i = 0; i < colorMap.length - 1; i++) {
                const c1 = colorMap[i];
                const c2 = colorMap[i + 1];
                if (temp >= c1.temp && temp <= c2.temp) {
                    const ratio = (temp - c1.temp) / (c2.temp - c1.temp);
                    const r = c1.color.r * (1 - ratio) + c2.color.r * ratio;
                    const g = c1.color.g * (1 - ratio) + c2.color.g * ratio;
                    const b = c1.color.b * (1 - ratio) + c2.color.b * ratio;
                    return new THREE.Color(r, g, b);
                }
            }
            return new THREE.Color(0xffffff); // Cor padrão se algo der errado
        }

        // Função para criar os cabos e pontos de temperatura
        // Esta função agora SEMPRE usará o objeto de dados que lhe for passado.
        function createSiloCables(dataToRender, siloHeightOffset) {
            const distribuicaoCabos = dataToRender.distribuicaoCabos;
            const alturaCabos = dataToRender.alturaCabos;
            const leiturasTemperatura = dataToRender.leiturasTemperatura;

            let currentCableIndex = 0;
            
            for (let i = 1; i <= 5; i++) {
                const numCablesInLine = distribuicaoCabos[`linha_${i}`] || 0;
                
                if (numCablesInLine > 0) {
                    let cableRadius;
                    if (i === 1) { // Linha 1 (central)
                        cableRadius = 0;
                    } else { // Outras linhas formam anéis
                        cableRadius = (i - 1) * configuracao.distanciaEntreAneisDeCabos;
                    }
                    
                    const effectiveNumCablesInLine = (cableRadius === 0 && numCablesInLine === 1) ? 1 : numCablesInLine;
                    const angleStep = (2 * Math.PI) / effectiveNumCablesInLine;

                    for (let j = 0; j < numCablesInLine; j++) {
                        currentCableIndex++;
                        let cableX, cableZ;

                        if (cableRadius === 0) { // Cabo central
                            cableX = 0;
                            cableZ = 0;
                        } else { // Cabos em anéis
                            const cableAngle = j * angleStep;
                            cableX = Math.cos(cableAngle) * cableRadius;
                            cableZ = Math.sin(cableAngle) * cableRadius;
                        }

                        const numPoints = alturaCabos[`cabo_${currentCableIndex}`] || 0;
                        const temperatures = leiturasTemperatura[`cabo_${currentCableIndex}`] || [];

                        // Cria o cabo (linha)
                        const cableMaterial = new THREE.LineBasicMaterial({ color: 0x888888 });
                        const cableGeometry = new THREE.BufferGeometry().setFromPoints([
                            new THREE.Vector3(cableX, siloHeightOffset, cableZ),
                            new THREE.Vector3(cableX, siloHeightOffset + numPoints * pointSpacing, cableZ)
                        ]);
                        const cableLine = new THREE.Line(cableGeometry, cableMaterial);
                        scene.add(cableLine);
                        objectsToRemove.push(cableLine);

                        // Cria os pontos de temperatura
                        for (let k = 0; k < numPoints; k++) {
                            const temp = temperatures[k] !== undefined ? temperatures[k] : 0;
                            const pointY = siloHeightOffset + k * pointSpacing;
                            const pointColor = getColorForTemperature(temp);

                            const pointGeometry = new THREE.SphereGeometry(configuracao.diametroDeExibicaoDaTemperatura / 2, 16, 16);
                            const pointMaterial = new THREE.MeshBasicMaterial({ color: pointColor });
                            const pointMesh = new THREE.Mesh(pointGeometry, pointMaterial);
                            pointMesh.position.set(cableX, pointY, cableZ);
                            scene.add(pointMesh);
                            objectsToRemove.push(pointMesh);

                            // Adicionar texto da temperatura
                            if (loadedFont && temp !== 0) {
                                const textMaterial = new THREE.MeshBasicMaterial({ color: 0x000000 });
                                const textGeometry = new THREE.TextGeometry(temp.toString(), {
                                    font: loadedFont,
                                    size: configuracao.tamanhoDaFonteDeTemperatura,
                                    height: 0.1,
                                });
                                textGeometry.computeBoundingBox();
                                const textWidth = textGeometry.boundingBox.max.x - textGeometry.boundingBox.min.x;
                                const textMesh = new THREE.Mesh(textGeometry, textMaterial);
                                textMesh.position.set(cableX + configuracao.diametroDeExibicaoDaTemperatura / 2 + 0.1, pointY, cableZ);
                                objectsToRemove.push(textMesh);
                                scene.add(textMesh);
                            }
                        }
                    }
                }
            }
        }


        // ==========================================================
        //  FUNÇÕES DE INICIALIZAÇÃO E RENDERIZAÇÃO
        // ==========================================================

        function init() {
            // Cena
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0xf0f0f0);

            // Câmera
            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.set(20, 20, 20);

            // Renderizador
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            document.body.appendChild(renderer.domElement);

            // Controles de Órbita
            controls = new THREE.OrbitControls(camera, renderer.domElement); 
            controls.enableDamping = true;
            controls.dampingFactor = 0.25;
            controls.screenSpacePanning = false;
            controls.maxPolarAngle = Math.PI / 2;

            // Luz Ambiente
            const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
            scene.add(ambientLight);

            // Luz Direcional
            const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
            directionalLight.position.set(5, 10, 7);
            scene.add(directionalLight);

            // PointLight para clarear o interior
            const pointLight = new THREE.PointLight(0xffffff, 1.0, 100);
            pointLight.position.set(0, 10, 0);
            scene.add(pointLight);

            // Carregar a fonte
            const fontLoader = new THREE.FontLoader();
            fontLoader.load('https://cdn.jsdelivr.net/npm/three@0.128.0/examples/fonts/helvetiker_regular.typeface.json', function (font) {
                loadedFont = font;
                console.log("Fonte carregada.");
                // Na inicialização, usamos os dados default para que algo apareça
                updateGraphData(JSON.stringify(defaultInitialData)); 
            });
            
            // Variáveis globais para as dimensões do silo e offset inicial
            // Serão preenchidas ou atualizadas em updateGraphData
            let initialSiloRadius = 7; // Valor inicial para garantir que o silo seja visível antes dos dados
            const basePlateHeight = 1;

            // Material do silo (cilindro e cone superior)
            const siloMaterial = new THREE.MeshPhysicalMaterial({ 
                color: 0xe0e0e0,
                metalness: 0.6,
                roughness: 0.3,
                transparent: true,
                opacity: 0.15,
                side: THREE.DoubleSide
            });

            // Placa da Base do Silo - Criada apenas UMA VEZ na inicialização
            siloBasePlate = new THREE.Mesh(
                new THREE.CylinderGeometry(initialSiloRadius + 1, initialSiloRadius + 1, basePlateHeight, 32),
                new THREE.MeshStandardMaterial({ color: 0x999999, roughness: 0.7, metalness: 0.1 })
            );
            siloBasePlate.position.y = 0;
            siloBasePlate.name = 'siloBasePlate';
            scene.add(siloBasePlate);

            // Criar o silo principal (cilindro e cone superior) com valores iniciais
            siloMesh = new THREE.Mesh(new THREE.CylinderGeometry(initialSiloRadius, initialSiloRadius, 15, 32), siloMaterial);
            siloMesh.name = 'siloCylinder';
            scene.add(siloMesh);

            siloTopCone = new THREE.Mesh(new THREE.ConeGeometry(initialSiloRadius, coneHeight, 32), siloMaterial);
            siloTopCone.name = 'siloTopCone';
            scene.add(siloTopCone);

            axesHelper = new THREE.AxesHelper(Math.max(initialSiloRadius * 1.5, 15 + coneHeight));
            axesHelper.position.y = 0;
            scene.add(axesHelper);
            
            animate();
        }

        // Função de animação
        function animate() {
            requestAnimationFrame(animate);
            controls.update();
            renderer.render(scene, camera);
        }

        // Função para atualizar o gráfico com novos dados
        window.updateGraphData = function(data) {
            try {
                const cleanedDataString = data.replace(/^"|"$/g, ''); 
                const parsedData = JSON.parse(cleanedDataString); 
                console.log("Dados recebidos do FlutterFlow (parsed):", parsedData);

                // Remover APENAS cabos, pontos e textos antigos
                const currentObjectsToRemove = [...objectsToRemove];
                currentObjectsToRemove.forEach(obj => {
                    scene.remove(obj);
                    if (obj.geometry) obj.geometry.dispose();
                    if (obj.material) {
                        if (Array.isArray(obj.material)) {
                            obj.material.forEach(mat => mat.dispose());
                        } else {
                            obj.material.dispose();
                        }
                    }
                });
                objectsToRemove = []; // Limpa o array principal após remover todos os objetos dinâmicos
                
                // Re-obter o material do silo (para garantir que não foi descartado)
                const siloMaterial = new THREE.MeshPhysicalMaterial({ 
                    color: 0xe0e0e0,
                    metalness: 0.6,
                    roughness: 0.3,
                    transparent: true,
                    opacity: 0.15,
                    side: THREE.DoubleSide
                });

                // --- AGORA TODOS OS CÁLCULOS ABAIXO USARÃO parsedData DIRETAMENTE ---
                let newMaxCableHeight = 0;
                let numberOfActiveLines = 0;

                // Loop para calcular numberOfActiveLines a partir dos DADOS RECEBIDOS
                for (let i = 1; i <= 5; i++) {
                    const numCablesInLine = parsedData.distribuicaoCabos[`linha_${i}`] || 0;
                    if (numCablesInLine > 0) {
                        numberOfActiveLines++;
                    }
                }

                let currentCableIndexCheck = 0;
                // Loop para calcular newMaxCableHeight a partir dos DADOS RECEBIDOS
                for (let i = 1; i <= 5; i++) {
                    const numCablesInLine = parsedData.distribuicaoCabos[`linha_${i}`] || 0;
                    for (let j = 0; j < numCablesInLine; j++) {
                        currentCableIndexCheck++;
                        const alturaDoCabo = parsedData.alturaCabos[`cabo_${currentCableIndexCheck}`] || 0;
                        if (alturaDoCabo > newMaxCableHeight) {
                            newMaxCableHeight = alturaDoCabo;
                        }
                    }
                }
                
                // Garante que a altura mínima do cilindro seja razoável
                if (newMaxCableHeight === 0) newMaxCableHeight = 10; 

                const basePlateHeight = 1;
                const cylinderOffsetFromBase = basePlateHeight;
                const newSiloCylinderHeight = newMaxCableHeight * pointSpacing;

                let currentSiloRadius = 0;
                if (numberOfActiveLines === 0) {
                    currentSiloRadius = configuracao.distanciaDoUltimoAnelDaParede;
                } else if (numberOfActiveLines === 1 && parsedData.distribuicaoCabos["linha_1"] === 1) {
                    currentSiloRadius = configuracao.distanciaDoUltimoAnelDaParede;
                }
                 else {
                    currentSiloRadius = (numberOfActiveLines - 1) * configuracao.distanciaEntreAneisDeCabos + configuracao.distanciaDoUltimoAnelDaParede;
                }

                // Atualiza o cilindro do silo
                siloMesh.geometry.dispose();
                siloMesh.geometry = new THREE.CylinderGeometry(currentSiloRadius, currentSiloRadius, newSiloCylinderHeight, 32);
                siloMesh.position.y = (newSiloCylinderHeight / 2) + basePlateHeight;
                siloMesh.material = siloMaterial;

                // Atualiza o cone superior
                siloTopCone.geometry.dispose();
                siloTopCone.geometry = new THREE.ConeGeometry(currentSiloRadius, coneHeight, 32);
                siloTopCone.position.y = newSiloCylinderHeight + basePlateHeight + (coneHeight / 2);
                siloTopCone.material = siloMaterial;

                // Atualiza a placa da base
                siloBasePlate.geometry.dispose();
                siloBasePlate.geometry = new THREE.CylinderGeometry(currentSiloRadius + 1, currentSiloRadius + 1, basePlateHeight, 32);
                siloBasePlate.position.y = 0;
                
                // CHAMA createSiloCables PASSANDO OS DADOS REAIS DO FLUTTERFLOW (parsedData)
                createSiloCables(parsedData, cylinderOffsetFromBase);

                // A altura total visual do silo (para o AxesHelper e câmera)
                const totalSiloVisualHeight = newSiloCylinderHeight + coneHeight + basePlateHeight;

                // Atualiza o AxesHelper
                if (axesHelper) {
                    scene.remove(axesHelper);
                    axesHelper.geometry.dispose();
                }
                axesHelper = new THREE.AxesHelper(Math.max(currentSiloRadius * 1.5, totalSiloVisualHeight));
                axesHelper.position.y = totalSiloVisualHeight / 2 - basePlateHeight;
                scene.add(axesHelper);

                // Ajusta o target dos controles e a posição da câmera
                controls.target.set(0, totalSiloVisualHeight / 2, 0);
                camera.position.set(currentSiloRadius * 2.5, controls.target.y + currentSiloRadius, currentSiloRadius * 2.5);
                controls.update();
                
            } catch (e) {
                console.error("Erro ao parsear dados JSON do FlutterFlow ou durante a atualização do gráfico:", e, "Dados originais recebidos:", data);
            }
        }; // Fim de window.updateGraphData

        init();

    </script>
</body>
</html>
