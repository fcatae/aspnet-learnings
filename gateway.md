A camada Web (frontend) publicando uma API de backend possui um código similar a um gateway:

    public string All()
    {
        HttpClient client = new HttpClient();
        var response = client.GetStringAsync("http://localhost:5001/api/tasks").Result;
        return response;
    }
    
O código é simples e pode ser parametrizado. Entretanto, é importante notar que o frontend possui 
uma camada anterior de proxy-reverso (IIS/Nginx) para realizar as tarefas HTTP de SSL e Compression.
