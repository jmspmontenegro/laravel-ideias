# Como implementar broadcast via Telegram

## 📌 Objetivo

A ideia é que o Laravel possa enviar comunicações para um gurpo do Telegram para avisos, erros, etc. Um bom uso é notificar exceptions da aplicação.

- [Documentação API Telegram](https://core.telegram.org/bots/api)

## 🚩 Pré-requisitos

- Criar um bot no Telegram. Acredite, isso é fácil! Abra o Telegram e inicie uma conversa com o @BotFathere ele vai ajudar com um passo a passo bem tranquilo. Ao final ele vai salvar um TOKEN que será usado na autenticação.

- Antes de mais nada eu gosto de sempre que uso APIs criar um repositório via Insominia no github e salvar as requests que vou usando. Fica a dica.

- No telegram, crie um grupo com o bot que você criou. Esse será o grupo de broadcast que enviará todas as mensagens. Assim basta adicionar mais pessoas no grupo para que possam receber as notificações.

## 🚧 Vamos lá!


### Criando arquivo config/api.php

Para comunicações via API, ou conexões externas eu costumo criar um arquivo no diretório config do Laravel para predefinir as configurações relativas à conexões. Neste caso, podemos criar um arquivo config/api.php onde teremos todas as configurações. 

Arquivo: config/api.php
```php
<?php

set_time_limit(1800);
    /*
    |--------------------------------------------------------------------------
    | Configurações dos WebServices/APIs
    |--------------------------------------------------------------------------
    |
    | Aqui estão especificadas as configurações necessárias para o consumo dos
    | webservices.
    |
     */
return [
    
    'telegram' => [
        'urlBase' => 'https://api.telegram.org/bot',
        'token' => 'bot123456789:jbd78sadvbdy63d37gda37bd8'  //Digite aqui o token que recebeu ao criar o bot lá no telegram com ajuda do @BOTFather.      
    ],
];
```

### Buscando o id do chat

Precisamos identificar qual é o **chat_id** do grupo que criamos nos pré-requisitos. Para isso basta executar a requisição abaixo. (Pode ser direto no navegador, ou se seguir minha sugestão crie a request no Insomnia e já deixe salvo em um repositório do github, isso costuma ser super útil).

https://api.telegram.org/botSEU_TOKEN/getUpdates

onde no lugar de "SEU_TOKEN" coloque aquele token salvo no arquio api.php.
Exemplo:
https://api.telegram.org/bot123456789:jbd78sadvbdy63d37gda37bd8/getUpdates

O resultado informará o chat_id:
```json
{
    "update_id":8393,
    "message":{
        "message_id":3,
        "from":{
            "id":7474,
            "first_name":"AAA"
        },
        "chat":{
            "id":ID_DO_SEU_GRUPO, // <-- Este é o id que você vai utilizar
            "title":""
        },
        "date":25497,
        "new_chat_participant":{
            "id":71,
            "first_name":"NAME",
            "username":"NOME_DO_SEU_BOT"
        }
    }
}
```

### Criando o serviço TelegramBot.php

Para facilitar o consumo, optei por criar um serviço para envio de dados da api do Telegram:

Arquivo: app/Http/Services/Telegram/TelegramBot.php
```php
<?php
namespace App\Http\Services\Telegram;

class TelegramBot
{
    /**
     * Url base para os webservices
     *
     * @var string
     */
    protected $urlBase;

    /**
     * Token de autenticação
     * @var string
     */
    protected $token;

    /**
     * Criando uma nova instancia do serviço
     *
     * @return void
     */
    public function __construct()
    {
        $configuracao = \Config::get('api.telegram');
        
        $this->urlBase = $configuracao['urlBase'];
        $this->token = $configuracao['token'];
    }

    /**
     * Envio de Broadtas em grupo de notificações
     */
    public static function sendBroadcast($message)
    {
        $telegramBot = new self();
        $url = $telegramBot->urlBase . $telegramBot->token . '/sendMessage';

        $client = new \GuzzleHttp\Client();

        $response = $client->request('POST', $url, ['query' => [
            'chat_id' => '-999999999',  //Este é o chat_id que pegou anteriormente
            'text' => $message,
        ]]);

        $statusCode = $response->getStatusCode();
        $content = $response->getBody();

        //Não esqueça de criar algumas regras de validação utilizando o $statusCode e o $content
        //Como envolvia algumas regras de negócio, aqui nesse repositório resolvi não incluir.
    }
}
```

### Como utilizar

Abaixo uma classe genérica beeeeeem simples que escrevi aqui apenas para demonstrar como usa o bot. Me baseei em um Job que de tempos em tempos importa dados de outra plataforma. Que é o conceito que usei incialmente esse BOT.

Mas a ideia é apenas demonstrar a chamada do BOT, porque isso pode ser usado em qualquer outra classe da aplicação. Inclusive uma dica legal é a possibilidade de implementar para que seja notificadas TODAS Expcetions. Isso é possível no laravel utilizando o Exception Handler (documentação: [https://laravel.com/docs/9.x/errors]).

```php
<?php

namespace  App\Console\Commands\MeuJob;

use App\Console\Commands\cronEssentials;
use App\Http\Services\Telegram\TelegramBot;

/**
 * Este é um Job que a cada meia hora importa dados de outra plataforma. O uso aqui
 * seria informar no grupo do telegram sempre que houver algum erro de importação.
 * Mas não precisa ser um Job, pode ser qualquer classe da aplicação :)
 */
class JobImportacao extends cronEssentials
{
     /**
     * Nome e assinatura do comando do console
     *
     * @var string
     */
    protected $signature = 'JOB:MeuJob';

    /**
     * Descrição do comando do console
     *
     * @var string
     */
    protected $description = 'Faz importação e dados de outra plataforma';

     /**
     * __Construct
     *
     * @return void
     */
    public function __construct()
    {
        parent::__construct();
    }

    /**
     * Execução do Job
     *
     * @return mixed
     */
    public function handle()
    {
        $importation = new ImportacaoDados();

        //Valida se houve algum erro na importação
        if($importation->getError()){
            TelegramBot::sendBroadcast('ERRO/Commands/JOB/JobImportacao:ImportacaoDados. Mensagem: ' . $importation->getErrorMessage());
        }
    }

}
```
