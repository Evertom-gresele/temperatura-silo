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
        // ==========================================================
        //  VARIÁVEIS GLOBAIS E FUNÇÕES AUXILIARES
        // ==========================================================

        let scene, camera, renderer, controls;
        let siloMesh, siloBaseCone, siloTopCone;
        let axesHelper;
        let loadedFont; // Variável para armazenar a fonte carregada
        let objectsToRemove = []; // Array para armazenar objetos a serem removidos na atualização

        // Configurações do silo
        const cableSpacing = 2; // Espaçamento entre cabos na circunferência
        const pointSpacing = 1; // Espaçamento vertical entre pontos de temperatura
        const coneHeight = 3; // Altura dos cones superior e inferior (base do silo está em y=0)

        // Mapa de cores para interpolação de temperaturas
        const colorMap = [
            { temp: 10, color: new THREE.Color(0x0000ff) }, // Azul (frio)
            { temp: 20, color: new THREE.Color(0x00ffff) }, // Ciano
            { temp: 30, color: new THREE.Color(0x00ff00) }, // Verde
            { temp: 40, color: new THREE.Color(0xffff00) }, // Amarelo
            { temp: 50, color: new THREE.Color(0xffa500) }, // Laranja
            { temp: 60, color: new THREE.Color(0xff0000) }  // Vermelho (quente)
        ];

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
        function createSiloCables(parsedData) { // Agora espera parsedData
            const distribuicaoCabos = parsedData.distribuicaoCabos;
            const alturaCabos = parsedData.alturaCabos;
            const leiturasTemperatura = parsedData.leiturasTemperatura;

            let currentCableIndex = 0; // Índice global do cabo
            let newSiloRadius = 0; // Raio do silo com base nos cabos

            // Calcular o novo raio do silo baseado no número de cabos
            let maxCablesInAnyLine = 0;
            for (const lineKey in distribuicaoCabos) {
                const numCablesInLine = distribuicaoCabos[lineKey];
                if (numCablesInLine > maxCablesInAnyLine) {
                    maxCablesInAnyLine = numCablesInLine;
                }
            }
            // Se não houver cabos, defina um raio padrão para evitar divisão por zero
            newSiloRadius = maxCablesInAnyLine > 0 ? (maxCablesInAnyLine * cableSpacing) / (2 * Math.PI) : 5; 

            // Criar cabos para cada linha
            for (let i = 1; i <= 5; i++) { // Loop para linha_1 a linha_5
                const numCablesInLine = distribuicaoCabos[`linha_${i}`] || 0;
                
                if (numCablesInLine > 0) {
                    // Calcula o ângulo inicial e o passo angular para esta linha
                    const startAngle = i * (Math.PI / 6); // Exemplo: cada linha começa em um ângulo diferente
                    const angleStep = (2 * Math.PI) / numCablesInLine;

                    for (let j = 0; j < numCablesInLine; j++) {
                        currentCableIndex++;
                        const cableAngle = startAngle + j * angleStep;
                        const cableX = Math.cos(cableAngle) * newSiloRadius;
                        const cableZ = Math.sin(cableAngle) * newSiloRadius;

                        const numPoints = alturaCabos[`cabo_${currentCableIndex}`] || 0;
                        const temperatures = leiturasTemperatura[`cabo_${currentCableIndex}`] || [];

                        // Cria o cabo (linha)
                        const cableMaterial = new THREE.LineBasicMaterial({ color: 0x888888 });
                        const cableGeometry = new THREE.BufferGeometry().setFromPoints([
                            new THREE.Vector3(cableX, 0, cableZ),
                            new THREE.Vector3(cableX, numPoints * pointSpacing, cableZ)
                        ]);
                        const cableLine = new THREE.Line(cableGeometry, cableMaterial);
                        scene.add(cableLine);
                        objectsToRemove.push(cableLine); // Adiciona para remoção posterior

                        // Cria os pontos de temperatura
                        for (let k = 0; k < numPoints; k++) {
                            const temp = temperatures[k] !== undefined ? temperatures[k] : 0;
                            const pointY = k * pointSpacing;
                            const pointColor = getColorForTemperature(temp);

                            const pointGeometry = new THREE.SphereGeometry(0.3, 16, 16);
                            const pointMaterial = new THREE.MeshBasicMaterial({ color: pointColor });
                            const pointMesh = new THREE.Mesh(pointGeometry, pointMaterial);
                            pointMesh.position.set(cableX, pointY, cableZ);
                            scene.add(pointMesh);
                            objectsToRemove.push(pointMesh); // Adiciona para remoção posterior

                            // Adicionar texto da temperatura
                            if (loadedFont && temp !== 0) { // Só adiciona se a fonte estiver carregada e temp não for 0
                                const textMaterial = new THREE.MeshBasicMaterial({ color: 0x000000 });
                                const textGeometry = new THREE.TextGeometry(temp.toString(), {
                                    font: loadedFont,
                                    size: 0.5,
                                    height: 0.1,
                                });
                                textGeometry.computeBoundingBox();
                                const textWidth = textGeometry.boundingBox.max.x - textGeometry.boundingBox.min.x;
                                const textMesh = new THREE.Mesh(textGeometry, textMaterial);
                                textMesh.position.set(cableX + 0.5, pointY, cableZ); // Posição ao lado do ponto
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
            scene.background = new THREE.Color(0xf0f0f0); // Cor de fundo cinza claro

            // Câmera
            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.set(20, 20, 20); // Posição inicial da câmera

            // Renderizador
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            document.body.appendChild(renderer.domElement);

            // Controles de Órbita
            controls = new OrbitControls(camera, renderer.domElement);
            controls.enableDamping = true; // para um movimento mais suave
            controls.dampingFactor = 0.25;
            controls.screenSpacePanning = false;
            controls.maxPolarAngle = Math.PI / 2; // Limita a rotação para não ir abaixo do chão

            // Luz Ambiente
            const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
            scene.add(ambientLight);

            // Luz Direcional
            const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
            directionalLight.position.set(5, 10, 7);
            scene.add(directionalLight);

            // Carregar a fonte (apenas uma vez na inicialização)
            const fontLoader = new THREE.FontLoader();
            fontLoader.load('https://unpkg.com/three@0.128.0/examples/fonts/helvetiker_regular.typeface.json', function (font) {
                loadedFont = font;
                console.log("Fonte carregada.");
            });
            
            // *** DESENHA UM SILO PADRÃO NA INICIALIZAÇÃO ***
            // Isso garante que siloMesh, siloBaseCone, siloTopCone e axesHelper
            // sejam definidos e adicionados à cena antes de updateGraphData ser chamado.
            const initialRadius = 5; // Raio padrão
            const initialCylinderHeight = 10; // Altura do cilindro padrão (sem os cones)
            const siloMaterial = new THREE.MeshStandardMaterial({ color: 0xcccccc, transparent: true, opacity: 0.7 });

            // Cilindro do silo
            siloMesh = new THREE.Mesh(new THREE.CylinderGeometry(initialRadius, initialRadius, initialCylinderHeight, 32), siloMaterial);
            siloMesh.position.y = initialCylinderHeight / 2; // Centro do cilindro na metade da altura
            siloMesh.name = 'siloCylinder';
            scene.add(siloMesh);
            objectsToRemove.push(siloMesh);

            // Cone inferior
            siloBaseCone = new THREE.Mesh(new THREE.ConeGeometry(initialRadius, coneHeight, 32), siloMaterial);
            siloBaseCone.position.y = -coneHeight / 2; // Abaixo da base do cilindro (y=0)
            siloBaseCone.rotation.x = Math.PI;
            siloBaseCone.name = 'siloBaseCone';
            scene.add(siloBaseCone);
            objectsToRemove.push(siloBaseCone);

            // Cone superior
            siloTopCone = new THREE.Mesh(new THREE.ConeGeometry(initialRadius, coneHeight, 32), siloMaterial);
            siloTopCone.position.y = initialCylinderHeight + (coneHeight / 2); // Acima do topo do cilindro
            siloTopCone.name = 'siloTopCone';
            scene.add(siloTopCone);
            objectsToRemove.push(siloTopCone);

            // AxesHelper
            axesHelper = new THREE.AxesHelper(Math.max(initialRadius, initialCylinderHeight / 2 + coneHeight)); // Ajusta o tamanho
            axesHelper.position.y = 0; // Na base do silo
            scene.add(axesHelper);
            objectsToRemove.push(axesHelper);
            // *** FIM DA ADIÇÃO DE CÓDIGO ***

            // Chame a animação
            animate();
        }

        // Função de animação
        function animate() {
            requestAnimationFrame(animate);
            controls.update(); // apenas necessário se enableDamping estiver ativado
            renderer.render(scene, camera);
        }

        // Função para atualizar o gráfico com novos dados
        window.updateGraphData = function(data) {
            try {
                // Tenta remover as aspas extras que o FlutterFlow pode adicionar
                const cleanedDataString = data.replace(/^"|"$/g, ''); 
                const parsedData = JSON.parse(cleanedDataString); 
                console.log("Dados recebidos do FlutterFlow (parsed):", parsedData); // Para depuração

                // 1. Remover objetos antigos (cabos, pontos, textos, e o AxesHelper/siloMesh se eles mudarem de tamanho)
                objectsToRemove.forEach(obj => {
                    scene.remove(obj);
                    // Dispose de geometria e material para liberar memória
                    if (obj.geometry) obj.geometry.dispose();
                    if (obj.material) {
                        if (Array.isArray(obj.material)) {
                            obj.material.forEach(mat => mat.dispose());
                        } else {
                            obj.material.dispose();
                        }
                    }
                });
                objectsToRemove = []; // Limpa o array

                // Recria o material do silo, caso tenha sido disposto
                const siloMaterial = new THREE.MeshStandardMaterial({ color: 0xcccccc, transparent: true, opacity: 0.7 });

                // Obter novo raio e altura a partir dos dados para o silo
                let newMaxCableHeight = 0; 
                let newSiloRadius = 0;

                // Calcular o novo raio do silo baseado no número de cabos em distribuicaoCabos
                let maxCablesInAnyLine = 0;
                for (const lineKey in parsedData.distribuicaoCabos) {
                    const numCablesInLine = parsedData.distribuicaoCabos[lineKey];
                    if (numCablesInLine > maxCablesInAnyLine) {
                        maxCablesInAnyLine = numCablesInLine;
                    }
                }
                newSiloRadius = maxCablesInAnyLine > 0 ? (maxCablesInAnyLine * cableSpacing) / (2 * Math.PI) : 5; 

                // Determinar a altura máxima dos cabos
                for (const caboKey in parsedData.alturaCabos) {
                    const alturaDoCabo = parsedData.alturaCabos[caboKey];
                     if (alturaDoCabo > newMaxCableHeight) {
                        newMaxCableHeight = alturaDoCabo;
                    }
                }
                if (newMaxCableHeight === 0) newMaxCableHeight = 10; // Altura mínima para o cilindro do silo

                const newCylinderHeight = newMaxCableHeight * pointSpacing; // Altura do cilindro principal

                // Atualiza/Cria o cilindro do silo
                // A lógica `if (siloMesh)` aqui se refere à variável global.
                // Como ele já foi criado em `init()`, esta parte irá atualizar sua geometria e posição.
                if (siloMesh) {
                    siloMesh.geometry.dispose();
                    siloMesh.geometry = new THREE.CylinderGeometry(newSiloRadius, newSiloRadius, newCylinderHeight, 32);
                    siloMesh.position.y = newCylinderHeight / 2; // Centro do cilindro
                    siloMesh.material = siloMaterial; // Garante que o material é o correto
                } else { // Caso não tenha sido inicializado (cenário menos provável agora)
                    siloMesh = new THREE.Mesh(new THREE.CylinderGeometry(newSiloRadius, newSiloRadius, newCylinderHeight, 32), siloMaterial);
                    siloMesh.position.y = newCylinderHeight / 2;
                    siloMesh.name = 'siloCylinder';
                    scene.add(siloMesh);
                }
                objectsToRemove.push(siloMesh); 

                // Atualiza/Cria o cone inferior
                if (siloBaseCone) {
                    siloBaseCone.geometry.dispose();
                    siloBaseCone.geometry = new THREE.ConeGeometry(newSiloRadius, coneHeight, 32);
                    siloBaseCone.position.y = -coneHeight / 2; // Posição abaixo da base do cilindro (y=0)
                    siloBaseCone.rotation.x = Math.PI;
                    siloBaseCone.material = siloMaterial;
                } else {
                    siloBaseCone = new THREE.Mesh(new THREE.ConeGeometry(newSiloRadius, coneHeight, 32), siloMaterial);
                    siloBaseCone.position.y = -coneHeight / 2;
                    siloBaseCone.rotation.x = Math.PI;
                    siloBaseCone.name = 'siloBaseCone';
                    scene.add(siloBaseCone);
                }
                objectsToRemove.push(siloBaseCone);

                // Atualiza/Cria o cone superior
                if (siloTopCone) {
                    siloTopCone.geometry.dispose();
                    siloTopCone.geometry = new THREE.ConeGeometry(newSiloRadius, coneHeight, 32);
                    siloTopCone.position.y = newCylinderHeight + (coneHeight / 2); // Posição acima do topo do cilindro
                    siloTopCone.material = siloMaterial;
                } else {
                    siloTopCone = new THREE.Mesh(new THREE.ConeGeometry(newSiloRadius, coneHeight, 32), siloMaterial);
                    siloTopCone.position.y = newCylinderHeight + (coneHeight / 2);
                    siloTopCone.name = 'siloTopCone';
                    scene.add(siloTopCone);
                }
                objectsToRemove.push(siloTopCone);

                // A altura total visual do silo (para o AxesHelper)
                const visualSiloTotalHeight = newCylinderHeight + (coneHeight * 2);

                // Atualiza/Cria o AxesHelper
                const visualAxesLength = Math.max(newSiloRadius, visualSiloTotalHeight / 2);
                if (axesHelper) {
                    scene.remove(axesHelper); 
                    axesHelper.geometry.dispose(); 
                    axesHelper = new THREE.AxesHelper(visualAxesLength);
                    axesHelper.position.y = 0; // Os eixos começam na base do silo (y=0)
                } else {
                    axesHelper = new THREE.AxesHelper(visualAxesLength);
                    axesHelper.position.y = 0;
                }
                scene.add(axesHelper); 
                objectsToRemove.push(axesHelper);

                // 4. Chamar createSiloCables com os NOVOS dados para desenhar os cabos, pontos e textos
                createSiloCables(parsedData); 

            } catch (e) {
                console.error("Erro ao parsear dados JSON do FlutterFlow ou durante a atualização do gráfico:", e, "Dados originais recebidos:", data);
            }
        }; // Fim de window.updateGraphData

        // Chame init() para configurar a cena e desenhar o silo e cabos iniciais
        init();

    </script>
</body>
</html>
