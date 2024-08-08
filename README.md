# Resolva hCaptcha em C++: Um Guia Completo


![](https://assets.capsolver.com/prod/images/post/2024-08-08/e232bdf8-fb76-4e77-9502-08e35c0facb1.png)

## Introdução

**hCaptcha** é um serviço CAPTCHA popular projetado para proteger sites contra bots e abuso automatizado. Ele apresenta aos usuários desafios que são fáceis para humanos resolverem, mas difíceis para sistemas automatizados. Esses desafios podem incluir identificar objetos em imagens ou resolver quebra-cabeças.

O principal objetivo do hCaptcha é garantir que as interações nos sites sejam realizadas por pessoas reais, não por scripts automatizados ou bots. Ele atua como um guardião para impedir submissões e interações automatizadas, melhorando tanto a segurança quanto a experiência do usuário.

## Por que Resolver hCaptcha com C++?

C++ é uma linguagem de programação poderosa, conhecida por seu desempenho e eficiência. Ela é comumente usada em cenários onde a velocidade e o gerenciamento de recursos são críticos, como no desenvolvimento de jogos, computação de alto desempenho e programação de sistemas. Aqui estão algumas razões pelas quais resolver o hCaptcha com C++ pode ser preferível:

1. **Desempenho**: C++ oferece controle detalhado sobre os recursos do sistema e pode executar tarefas com overhead mínimo, tornando-se adequado para cenários que exigem alto desempenho e velocidade.

2. **Integração com Sistemas Existentes**: Muitos sistemas e aplicações legadas são construídos usando C++. Se você está trabalhando em um ambiente desse tipo, usar C++ para resolver o hCaptcha pode ser uma escolha natural para manter a consistência.

3. **Controle de Baixo Nível**: C++ proporciona controle de baixo nível sobre hardware e recursos do sistema, o que pode ser vantajoso para criar soluções altamente otimizadas.

4. **Compatibilidade**: C++ pode se conectar com várias APIs e bibliotecas, possibilitando a integração de serviços de terceiros, como o CapSolver, para resolver CAPTCHAs.

## Visão Geral do Guia

Neste guia, exploraremos como resolver hCaptcha usando C++ interagindo com a [API CapSolver](https://www.capsolver.com/?utm_source=official&utm_medium=blog&utm_campaign=hc). Esse processo envolve criar uma tarefa para o desafio hCaptcha e, em seguida, recuperar o resultado dessa tarefa. Utilizaremos a biblioteca `cpr` para fazer solicitações HTTP e a biblioteca `jsoncpp` para analisar dados JSON.

Seguindo este tutorial, você aprenderá a:
1. Configurar um projeto C++ com as bibliotecas necessárias.
2. Criar uma tarefa para resolver um desafio hCaptcha.
3. Recuperar o resultado da tarefa e usá-lo em sua aplicação.

Seja integrando a solução do hCaptcha em uma aplicação C++ existente ou desenvolvendo uma nova ferramenta, este guia fornecerá o conhecimento e o código necessários para atingir seus objetivos de forma eficiente.

## Resolvendo hCaptcha em C++

### Pré-requisitos

Antes de começarmos, certifique-se de ter as seguintes bibliotecas instaladas:

1. **cpr**: Uma biblioteca HTTP para C++.
2. **jsoncpp**: Uma biblioteca C++ para análise de JSON.

Você pode instalá-las usando [vcpkg](https://github.com/microsoft/vcpkg):

```bash
vcpkg install cpr jsoncpp
```

### Passo 1: Configurando Seu Projeto

Crie um novo projeto C++ e inclua os cabeçalhos necessários para `cpr` e `jsoncpp`.

```cpp
#include <iostream>
#include <cpr/cpr.h>
#include <json/json.h>
#include <thread>
#include <chrono>
```

### Passo 2: Definir Funções para Criar e Obter Resultados de Tarefas

Definiremos duas funções principais: `createTask` e `getTaskResult`.

1. **createTask**: Esta função cria uma tarefa hCaptcha.
2. **getTaskResult**: Esta função recupera o resultado da tarefa criada.

Aqui está o código completo:

```cpp
#include <iostream>
#include <cpr/cpr.h>
#include <json/json.h>
#include <thread>
#include <chrono>

std::string createTask(const std::string& apiKey, const std::string& websiteURL, const std::string& websiteKey) {
    Json::Value requestBody;
    requestBody["clientKey"] = apiKey;
    requestBody["task"]["type"] = "HCaptchaTaskProxyless";
    requestBody["task"]["websiteURL"] = websiteURL;
    requestBody["task"]["websiteKey"] = websiteKey;

    Json::StreamWriterBuilder writer;
    std::string requestBodyStr = Json::writeString(writer, requestBody);

    cpr::Response response = cpr::Post(
        cpr::Url{"https://api.capsolver.com/createTask"},
        cpr::Body{requestBodyStr},
        cpr::Header{{"Content-Type", "application/json"}}
    );

    Json::CharReaderBuilder reader;
    Json::Value responseBody;
    std::string errs;
    std::istringstream s(response.text);
    std::string taskId;

    if (Json::parseFromStream(reader, s, &responseBody, &errs)) {
        if (responseBody["errorId"].asInt() == 0) {
            taskId = responseBody["taskId"].asString();
        } else {
            std::cerr << "Error: " << responseBody["errorCode"].asString() << std::endl;
        }
    } else {
        std::cerr << "Failed to parse response: " << errs << std::endl;
    }

    return taskId;
}

std::string getTaskResult(const std::string& apiKey, const std::string& taskId) {
    Json::Value requestBody;
    requestBody["clientKey"] = apiKey;
    requestBody["taskId"] = taskId;

    Json::StreamWriterBuilder writer;
    std::string requestBodyStr = Json::writeString(writer, requestBody);

    while (true) {
        cpr::Response response = cpr::Post(
            cpr::Url{"https://api.capsolver.com/getTaskResult"},
            cpr::Body{requestBodyStr},
            cpr::Header{{"Content-Type", "application/json"}}
        );

        Json::CharReaderBuilder reader;
        Json::Value responseBody;
        std::string errs;
        std::istringstream s(response.text);

        if (Json::parseFromStream(reader, s, &responseBody, &errs)) {
            if (responseBody["status"].asString() == "ready") {
                return responseBody["solution"]["gRecaptchaResponse"].asString();
            } else if (responseBody["status"].asString() == "processing") {
                std::cout << "Task is still processing, waiting for 5 seconds..." << std::endl;
                std::this_thread::sleep_for(std::chrono::seconds(5));
            } else {
                std::cerr << "Error: " << responseBody["errorCode"].asString() << std::endl;
                break;
            }
        } else {
            std::cerr << "Failed to parse response: " << errs << std::endl;
            break;
        }
    }

    return "";
}

int main() {
    std::string apiKey = "YOUR_API_KEY";
    std::string websiteURL = "https://example.com";
    std::string websiteKey = "SITE_KEY";

    std::string taskId = createTask(apiKey, websiteURL, websiteKey);
    if (!taskId.empty()) {
        std::cout << "Task created successfully. Task ID: " << taskId << std::endl;
        std::string hcaptchaResponse = getTaskResult(apiKey, taskId);
        std::cout << "hCaptcha Response: " << hcaptchaResponse << std::endl;
    } else {
        std::cerr << "Failed to create task." << std::endl;
    }

    return 0;
}
```

### Explicação

1. **Função createTask**: Esta função constrói um corpo de solicitação JSON com os parâmetros necessários (`apiKey`, `websiteURL`, `websiteKey`) e o envia para a API CapSolver para criar uma tarefa hCaptcha. Ela analisa a resposta para obter o `taskId`.

2. **Função getTaskResult**: Esta função verifica repetidamente o status da tarefa criada usando o `taskId` até que a tarefa seja concluída. Uma vez concluída, ela recupera e retorna a resposta do hCaptcha.

3. **Função main**: A função principal inicializa as variáveis necessárias (`apiKey`, `websiteURL`, `websiteKey`), chama `createTask` para obter um `taskId` e, em seguida, chama `getTaskResult` para obter a solução do hCaptcha.

### Conclusão

Este guia demonstrou como resolver hCaptcha em C++ usando a API CapSolver. Seguindo os passos acima, você pode integrar a solução do hCaptcha em suas aplicações C++. Certifique-se de tratar as chaves da API e outras informações sensíveis de maneira segura na sua implementação real.

Sinta-se à vontade para personalizar e expandir o código para atender aos seus requisitos específicos. Boa programação!
