<script>
    // ... seu código existente (handleWebViewJson, updateGraphData, etc.) ...

    // Função para solicitar dados do FlutterFlow
    async function requestDataFromFlutter() {
        if (window.flutter_inappwebview) {
            try {
                // Chama a Ação Personalizada 'getSiloJsonDataFromFlutter'
                // O nome da função aqui DEVE corresponder ao nome gerado pelo FlutterFlow para a Ação Personalizada
                // O FlutterFlow geralmente expõe as Ações Personalizadas como `flutter_inappwebview.callHandler('yourCustomActionName')`

                const jsonData = await window.flutter_inappwebview.callHandler('getSiloJsonDataFromFlutter');
                if (jsonData) {
                    console.log('Dados recebidos do FlutterFlow:', jsonData);
                    // Chama sua função para atualizar o gráfico com os dados recebidos
                    handleWebViewJson(jsonData); 
                } else {
                    console.log('Nenhum dado recebido do FlutterFlow.');
                }
            } catch (error) {
                console.error('Erro ao solicitar dados do FlutterFlow:', error);
            }
        } else {
            console.log('flutter_inappwebview não está disponível. Não é possível solicitar dados.');
        }
    }

    // Chama a função para solicitar dados assim que o WebView estiver pronto
    // Você pode chamar isso no final do seu script ou após um evento de carregamento
    document.addEventListener('DOMContentLoaded', () => {
         requestDataFromFlutter();
    });

    // Opcional: Se você quiser que o WebView puxe dados periodicamente
    // setInterval(requestDataFromFlutter, 5000); // A cada 5 segundos

</script>
