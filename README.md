<!DOCTYPE 6 html>
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
        const basePlateHeight = 1; // Altura da placa da base
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
        function createSiloCables(dataToRender, siloHeightOffset) {
            console.log("createSiloCables: Desenhando com os dados:", dataToRender);
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
                    
                    // Ajuste para garantir que a linha 1 sempre tenha um cabo, mesmo que distribuicaoCabos["linha_1"] seja 0 para o cálculo do ângulo
                    // Se numCablesInLine for 0, não deveria entrar aqui, então esta parte foi ajustada
                    // const effectiveNumCablesInLine = (cableRadius === 0 && numCablesInLine === 1) ? 1 : numCablesInLine;
                    const angleStep = (numCablesInLine > 1) ? (2 * Math.PI) / numCablesInLine : 0; // Evita divisão por zero se houver apenas 1 cabo na linha (ou nenhum)


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

                        // console.log(`Cabo ${currentCableIndex}: X=${cableX.toFixed(2)}, Y inicial=${siloHeightOffset}, Z=${cableZ.toFixed(2)}, Altura=${numPoints * pointSpacing}, Temps=${temperatures.length}`);

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
                            if (loadedFont && temp !== 0) { // Renderiza texto apenas se a temperatura não for 0
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
            // Use um raio inicial para que o silo seja visível
            const initialSiloRadiusForInit = 7; 
            siloBasePlate = new THREE.Mesh(
                new THREE.CylinderGeometry(initialSiloRadiusForInit + 1, initialSiloRadiusForInit + 1, basePlateHeight, 32),
                new THREE.MeshStandardMaterial({ color: 0x999999, roughness: 0.7, metalness: 0.1 })
            );
            siloBasePlate.position.y = 0;
            siloBasePlate.name = 'siloBasePlate';
            scene.add(siloBasePlate);

            // Criar o silo principal (cilindro e cone superior) com valores iniciais
            siloMesh = new THREE.Mesh(new THREE.CylinderGeometry(initialSiloRadiusForInit, initialSiloRadiusForInit, 15, 32), siloMaterial);
            siloMesh.name = 'siloCylinder';
            scene.add(siloMesh);

            siloTopCone = new THREE.Mesh(new THREE.ConeGeometry(initialSiloRadiusForInit, coneHeight, 32), siloMaterial);
            siloTopCone.name = 'siloTopCone';
            scene.add(siloTopCone);

            axesHelper = new THREE.AxesHelper(Math.max(initialSiloRadiusForInit * 1.5, 15 + coneHeight));
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
            console.log("updateGraphData: Função chamada com dados brutos:", data);
            try {
                // Remover aspas duplas extras se existirem
                const cleanedDataString = data.replace(/^"|"$/g, ''); 
                const parsedData = JSON.parse(cleanedDataString); 
                console.log("updateGraphData: Dados JSON parseados:", parsedData);

                // --- REMOVER OBJETOS ANTIGOS ---
                // É CRÍTICO que this.scene.children NÃO seja modificado durante a iteração
                // Criamos uma cópia para iterar
                const objectsToDispose = [];
                for (let i = scene.children.length - 1; i >= 0; i--) {
                    const obj = scene.children[i];
                    // Identificar apenas os objetos que foram adicionados dinamicamente (cabos, pontos, textos)
                    // Verifique se o objeto está no array objectsToRemove e remova-o da cena
                    if (objectsToRemove.includes(obj)) {
                        scene.remove(obj);
                        objectsToDispose.push(obj); // Adiciona para descarte posterior
                    }
                }
                objectsToRemove = []; // Limpa o array principal após remover todos os objetos dinâmicos
                
                // Descartar geometrias e materiais para liberar memória da GPU
                objectsToDispose.forEach(obj => {
                    if (obj.geometry) {
                        obj.geometry.dispose();
                        // console.log("Geometry disposed:", obj.geometry);
                    }
                    if (obj.material) {
                        if (Array.isArray(obj.material)) {
                            obj.material.forEach(mat => {
                                mat.dispose();
                                // console.log("Material array item disposed:", mat);
                            });
                        } else {
                            obj.material.dispose();
                            // console.log("Material disposed:", obj.material);
                        }
                    }
                });
                console.log("updateGraphData: Objetos antigos removidos e descartados.");

                // Re-obter o material do silo (para garantir que não foi descartado)
                const siloMaterial = new THREE.MeshPhysicalMaterial({ 
                    color: 0xe0e0e0,
                    metalness: 0.6,
                    roughness: 0.3,
                    transparent: true,
                    opacity: 0.15,
                    side: THREE.DoubleSide
                });

                // --- CÁLCULO DE DIMENSÕES DO SILO BASEADO NOS DADOS RECEBIDOS ---
                let newMaxCableHeight = 0;
                let numberOfActiveLines = 0;

                // Calcula numberOfActiveLines (quantos anéis/linhas têm cabos)
                for (let i = 1; i <= 5; i++) {
                    const numCablesInLine = parsedData.distribuicaoCabos[`linha_${i}`] || 0;
                    if (numCablesInLine > 0) {
                        numberOfActiveLines++;
                    }
                }
                console.log("updateGraphData: numberOfActiveLines (baseado em dados recebidos):", numberOfActiveLines);

                let currentCableIndexCheck = 0;
                // Calcula newMaxCableHeight (altura do cabo mais alto)
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
                console.log("updateGraphData: newMaxCableHeight (baseado em dados recebidos):", newMaxCableHeight);
                
                // Garante que a altura mínima do cilindro seja razoável para o silo ter volume
                if (newMaxCableHeight === 0) newMaxCableHeight = 1; // Para evitar altura zero, desenhe um silo mínimo

                const cylinderOffsetFromBase = basePlateHeight; // Cilindro começa logo acima da placa base
                const newSiloCylinderHeight = newMaxCableHeight * pointSpacing; // Altura do cilindro baseada na altura dos cabos
                console.log("updateGraphData: newSiloCylinderHeight calculada:", newSiloCylinderHeight);

                let currentSiloRadius = 0;
                if (numberOfActiveLines === 0) {
                    currentSiloRadius = configuracao.distanciaDoUltimoAnelDaParede; // Raio mínimo se não houver cabos
                } else if (numberOfActiveLines === 1 && parsedData.distribuicaoCabos["linha_1"] === 1) {
                    currentSiloRadius = configuracao.distanciaDoUltimoAnelDaParede; // Raio mínimo se apenas um cabo central
                } else {
                    currentSiloRadius = (numberOfActiveLines - 1) * configuracao.distanciaEntreAneisDeCabos + configuracao.distanciaDoUltimoAnelDaParede;
                }
                console.log("updateGraphData: currentSiloRadius calculada:", currentSiloRadius);

                // --- ATUALIZAÇÃO DAS GEOMETRIAS DOS OBJETOS DO SILO ---
                // Certifique-se de que as geometrias antigas são descartadas
                siloMesh.geometry.dispose();
                siloMesh.geometry = new THREE.CylinderGeometry(currentSiloRadius, currentSiloRadius, newSiloCylinderHeight, 32);
                siloMesh.position.y = (newSiloCylinderHeight / 2) + basePlateHeight;
                siloMesh.material = siloMaterial; // Reaplicar material por segurança
                console.log("updateGraphData: Silo cilíndrico atualizado.");

                siloTopCone.geometry.dispose();
                siloTopCone.geometry = new THREE.ConeGeometry(currentSiloRadius, coneHeight, 32);
                siloTopCone.position.y = newSiloCylinderHeight + basePlateHeight + (coneHeight / 2);
                siloTopCone.material = siloMaterial; // Reaplicar material por segurança
                console.log("updateGraphData: Silo cone atualizado.");

                siloBasePlate.geometry.dispose();
                siloBasePlate.geometry = new THREE.CylinderGeometry(currentSiloRadius + 1, currentSiloRadius + 1, basePlateHeight, 32);
                siloBasePlate.position.y = 0; // Posição fixa na base
                console.log("updateGraphData: Placa base atualizada.");
                
                // CHAMA createSiloCables PASSANDO OS DADOS REAIS DO FLUTTERFLOW (parsedData)
                createSiloCables(parsedData, cylinderOffsetFromBase);
                console.log("updateGraphData: createSiloCables chamado.");

                // A altura total visual do silo (para o AxesHelper e câmera)
                const totalSiloVisualHeight = newSiloCylinderHeight + coneHeight + basePlateHeight;
                console.log("updateGraphData: totalSiloVisualHeight:", totalSiloVisualHeight);

                // Atualiza o AxesHelper
                if (axesHelper) {
                    scene.remove(axesHelper);
                    axesHelper.geometry.dispose();
                    axesHelper.material.dispose();
                }
                axesHelper = new THREE.AxesHelper(Math.max(currentSiloRadius * 1.5, totalSiloVisualHeight));
                axesHelper.position.y = totalSiloVisualHeight / 2 - basePlateHeight; 
                scene.add(axesHelper);
                console.log("updateGraphData: AxesHelper atualizado.");

                // Ajusta o target dos controles e a posição da câmera
                controls.target.set(0, totalSiloVisualHeight / 2, 0); 
                camera.position.set(currentSiloRadius * 2.5, controls.target.y + currentSiloRadius, currentSiloRadius * 2.5);
                controls.update();
                console.log("updateGraphData: Câmera e controles atualizados.");
                
            } catch (e) {
                console.error("Erro ao parsear dados JSON do FlutterFlow ou durante a atualização do gráfico:", e, "Dados originais recebidos:", data);
            }
        }; // Fim de window.updateGraphData

        init();

        // Tratamento de redimensionamento da janela
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

    </script>
</body>
</html>
