# ChatApiWA-InstructionsCSharp
Данный репозиторий хранит в себе инструкцию по работе с API сайта chat-api.com. И webhook сервер.

# Создание WhatsApp бота с использованием chat-api.com на C#. Руководство

Функционал тестового бота будет ограничен следующими функциями:
* Реакция приветственным текстом на сообщение, команды которого нет у бота
* Отправка текущего chat id
* Отправка файлов различных форматов
* Отправка голосового сообщения
* Отправка геолокации
* Создание отдельной группы с собеседником и ботом

Писать наш бот будем с использованием технологии ASP.Net для поднятия сервера, который будет обрабатывать и отвечать на запросы пользователей. 

# Глава 1. Создание проекта ASP.Net
Откроем Visual Studio и создадим проект "Веб приложение ASP.NET Core".
![](https://i9.wampi.ru/2020/04/18/15c8869c253ee23a3.png)

Далее выберем шаблон с пустым проектом (также можно выбрать шаблон API, который будет включать в себя уже необходимые контроллеры, которые останется только отредактировать. Мы же для наглядности создадим все с нуля)
![](https://i9.wampi.ru/2020/04/18/240a49784641f7d39.png)

Откроем файл **Startup.cs** и впишем в метод Configure данный код:
```csharp
 public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
             if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }
            app.UseHttpsRedirection();
            app.UseRouting();
            app.UseAuthorization();
            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
            });
        }
```
Это позволит нам настроить навигацию с использованием контроллера.
Сейчас приступим к написанию самого контроллера.

Для этого создадим в проекте папку с названием Controllers, в которой создадим наш класс контроллер WebHookController
![](https://i9.wampi.ru/2020/04/18/3a80d3e82766b7722.png)

Наш контроллер должен наследовать класс **ControllerBase** и быть помечен атрибутами
```csharp
using Microsoft.AspNetCore.Mvc;
namespace WaBot.Controllers
{
    [ApiController]
    [Route("/")]
    public class WebHookController : ControllerBase
    {

    }
}
```
Атрибут **Route** отвечат за адрес, по которому будет срабатывать данный котроллер. Указываем базовый путь домена.

На данном этапе наш контоллер практически готов. Теперь нам необходимо добавить методы по работе с API WA и другие вспомогательные классы, которые пригодятся нам в работе.

# Глава 2. Класс API
В этой главе мы рассмотрим написание класса, который будет отвечать за взаимодействие с API сайта chat-api.com.
Документацию можно почитать [здесь](https://chat-api.com/ru/docs.html).
Создадим класс **WaApi**:
```csharp
 public class WaApi
    {
        private string APIUrl = "";
        private string token = "";

        public WaApi(string aPIUrl, string token)
        {
            APIUrl = aPIUrl;
            this.token = token;
        }
    }
```

Данный класс будет хранить в себе поля APIUrl и token, которые необходимы для работы с API. Получить их можно в личном кабинете оплаченных аккаунтов chat-api.com
![](https://i9.wampi.ru/2020/04/18/7eef658786e5950f9.png)

И конструктор, который присваивает значения в поля. Благодаря этому мы можем иметь несколько объектов, которые могут представлять различных ботов в случае, если необходимо настроить работу нескольких ботов одновременно.

### Метод для отправки запросов
Добавим в этот класс асинхронный метод, который будет осуществлять отправку POST запросов:
```csharp
  public async Task<string> SendRequest(string method, string data)
        {
            string url = $"{APIUrl}{method}?token={token}";

            using (var client = new HttpClient())
            {
                client.BaseAddress = new Uri(url);
                var content = new StringContent(data, Encoding.UTF8, "application/json");
                var result = await client.PostAsync("", content);
                return await result.Content.ReadAsStringAsync();
            }
        }
```
Данный метод принимает два аргумента.
+ *method* - название нужного метода согласно [документации](https://chat-api.com/ru/docs.html)
+ *data* - json строка для отправки

В методе сформируем строку url, на которую будет отправляться запрос. 
И далее делаем POST запрос  на данный адрес с помощью класса System.Net.Http.HttpClient
Возвращаем ответ сервера.

На основе данного метода мы можем сделать необходимый нам функционал для работы бота.

### Метод для отправки сообщений
```cshapr
public async Task<string> SendMessage(string chatID, string text)
        {
            var data = new Dictionary<string, string>()
            {
                {"chatId",chatID },
                { "body", text }
            };
            return await SendRequest("sendMessage", JsonConvert.SerializeObject(data));
        }
```
Данный метод  в качестве параметров принимает:
+ chatId - ID чата, куда необходимо отправить сообщение
+ text - текст отправляемого сообщения.

Для того, чтобы сформировать строку Json, воспользуемся удобной библиотекой [Newtonsoft.Json](https://www.newtonsoft.com/json).
Создадим словарь, ключом которого будет строка с необходимым Json полем, согласно [документации](https://chat-api.com/ru/docs.html), а значением - наши параметры.
Досточно просто вызвать метод **JsonConvert.SerializeObject** и передать в него наш словарь для формирования Json строки.
Вызываем метод **SendRequest**, передав в него название метода для отправки сообщений и нашу json строку. Таким образом бот будет отвечать пользователям.

### Метод для отправки голосового сообщения
```csharp 
 public async Task<string> SendOgg(string chatID)
        {
            string ogg = "https://firebasestorage.googleapis.com/v0/b/chat-api-com.appspot.com/o/audio_2019-02-02_00-50-42.ogg?alt=media&token=a563a0f7-116b-4606-9d7d-172426ede6d1";
            var data = new Dictionary<string, string>
            {
                {"audio", ogg },
                {"chatId", chatID }
            };

            return await SendRequest("sendAudio", JsonConvert.SerializeObject(data));
        }
```
Далее логика построения методов аналогична. Смотрим [документацию](https://chat-api.com/ru/docs.html) и отправляем на сервер нужные данные, вызывая необходимые методы. 
Для отправки голосового сообщения используется ссылка на файл формата .ogg и метод **sendAudio**

### Метод для отправки геолокации
```csharp
public async Task<string> SendGeo(string chatID)
        {
            var data = new Dictionary<string, string>()
            {
                { "lat", "55.756693" },
                { "lng", "37.621578" },
                { "address", "Ваш адрес" },
                { "chatId", chatID}
            };
            return await SendRequest("sendLocation", JsonConvert.SerializeObject(data));
        }
```

### Метод для создания группы
```csharp
  public async Task<string> CreateGroup(string author)
        {
            var phone = author.Replace("@c.us", "");
            var data = new Dictionary<string, string>()
            {
                { "groupName", "Группа C#"},
                { "phones", phone },
                { "messageText", "Это ваша группа." }
            };
            return await SendRequest("group", JsonConvert.SerializeObject(data));
        }
```

### Метод для отправки файлов
``` csharp
public async Task<string> SendFile(string chatID, string format)
        {
            var availableFormat = new Dictionary<string, string>()
            {
                {"doc", Base64String.Doc },
                {"gif",Base64String.Gif },

                { "jpg",Base64String.Jpg },
                { "png", Base64String.Png },
                { "pdf", Base64String.Pdf },
                { "mp4",Base64String.Mp4 },
                { "mp3", Base64String.Mp3}
            };

            if (availableFormat.ContainsKey(format))
            {
                var data = new Dictionary<string, string>(){
                    { "chatId", chatID },
                    { "body", availableFormat[format] },
                    { "filename", "yourfile" },
                    { "caption", $"Ваш файл" }                  
                };

                return await SendRequest("sendFile", JsonConvert.SerializeObject(data));
            }

            return await SendMessage(chatID, "Нет файла с таким форматом");
        }
```
Обговорим некоторые моменты, связанные с данным методом. 
Он принимает в параметры:
+ chatID - ID чата
+ format - формат файла, который необходимо отправить.

Для того, чтобы отправить файл, официальная [документация](https://chat-api.com/ru/docs.html) предусматривает несколько способов:
+ Ссылка на файл, который нужно отправить
+ Строка, которая является файлом, закодированным с помощью метода Base64.

**Рекомедуется отправлять файлы с помощью второго способа**, закодировав файлы в Base64 формат. В **Главе 4** я более подробно об этом напишу. А сейчас стоит знать, что я описал статический класс **Base64String**, в котором описал свойства, записав в них все тестовые файлы нужных форматов. В методе просто вызываю свойство нужного формата и передаю данную Base64 строку на сервер.

На этом базовый функционал нашего класса API описан. Теперь соединим наш контроллер из **Главы 1** и api.

### Глава 3. Обработка запросов. 
Вернемся к контроллеру из первой главы. 
Опишем метод внутри контроллера, который будет обрабатывать post-запросы, приходящие на наш сервер от chat-api.com.
Назовем метод Post(название может быть любым) и пометим его атрибутом [HttpPost], что будет означать реагирование на Post запросы:
``` csharp
        [HttpPost]
        public async Task<string> Post(Answer data)
        {
            return "";          
        } 
```
Принимать наш метод будет класс **Answer**, который является десериализированным объектом из пришедшей к нам строки json. 
Для того, чтобы описать класс **Answer**, нам потребуется узнать, какой json будет к нам приходить. Для этого можно воспользоваться удобным разделом **"Тестирование" - "Симуляция Webhooka"**

![](https://i9.wampi.ru/2020/04/18/455bfa4bc0a69e28a.png)
Справа мы можем видеть json тело, которое будет к нам приходить.
Воспользуемся сервисом [конвертации json в c#](http://json2csharp.com/).
Либо опишем класс сами, ипользуя атрибуты библиотеки [Newtonsoft.Json](https://www.newtonsoft.com/json):
```csharp 
  public partial class Answer
    {
        [JsonProperty("instanceId")]
        public string InstanceId { get; set; }

        [JsonProperty("messages")]
        public Message[] Messages { get; set; }
    }

    public partial class Message
    {
        [JsonProperty("id")]
        public string Id { get; set; }

        [JsonProperty("body")]
        public string Body { get; set; }

        [JsonProperty("type")]
        public string Type { get; set; }

        [JsonProperty("senderName")]
        public string SenderName { get; set; }

        [JsonProperty("fromMe")]
        public bool FromMe { get; set; }

        [JsonProperty("author")]
        public string Author { get; set; }

        [JsonProperty("time")]
        public long Time { get; set; }

        [JsonProperty("chatId")]
        public string ChatId { get; set; }

        [JsonProperty("messageNumber")]
        public long MessageNumber { get; set; }
    }
```
Теперь, когда у нас есть объектное представление пришедшего запроса, сделаем обработку его в контроллере.

Внутри контроллера создадим статическое поле, которым будет являться наш api ссылкой и токеном:
```cshapr
private static readonly WaApi api = new WaApi("https://eu115.chat-api.com/instance12345/", "123456789token");
```

В цикле метода проходимся по всем пришедшим к нам сообщениям и делаем проверку на то, что обрабатываемое сообщение не является нашим собственным. Это нужно для того, чтобы бот не зацикливался сам на себе. Если все же сообщение от самого себя - пропускаем:
```csharp 
        [HttpPost]
        public async Task<string> Post(Answer data)
        {
            foreach (var message in data.Messages)
            {
                if (message.FromMe)
                    continue;
            }
        }
```
Далее опишем switch, в который будем передавать получаемые команды. Применим к свойству Body метод Split(), чтобы разбить сообщение по пробелам. Передадим в switch первую команду, вызвав метод ToLower(), чтобы стиль написания команды не играл роли и обрабатывался одинаково:

```csharp
        [HttpPost]
        public async Task<string> Post(Answer data)
        {
            foreach (var message in data.Messages)
            {
                if (message.FromMe)
                    continue;

                switch (message.Body.Split()[0].ToLower())
                {
                    case "chatid":
                        return await api.SendMessage(message.ChatId, $"Ваш ID: {message.ChatId}");
                    case "file":
                        var texts = message.Body.Split();
                        if (texts.Length > 1)
                            return await api.SendFile(message.ChatId, texts[1]);
                        break;
                    case "ogg":
                        return await api.SendOgg(message.ChatId);
                    case "гео":
                        return await api.SendGeo(message.ChatId);
                    case "group":
                        return await api.CreateGroup(message.Author);
                    default:
                        return await api.SendMessage(message.ChatId, welcomeMessage);
                }             
            }
            return "";          
        }  
```
В **case** запишем все необходимые нам команды и будем вызывать методы из объекта нашего api, которые их реализауют.
**default** будет обрабатывать команды, которых не существует. Для этого просто будем отправлять сообщение из меню бота пользователю.
Всё, наш бот готов. Он уже может отвечать и обрабатывать команды пользователя, остается только добавить его на хостинг и указать домен в качестве webhook в личном кабинете пользователя chat-api.com.
![](https://i9.wampi.ru/2020/04/18/8909f12e10cb42fa7.png)
В следующих главах расскажу о base64 и возможных ошибках, с которыми вы можете столкнуться.

# Глава 4. Base64

**Base64** - это стандарт кодирования данных. С помощью данного стандарта мы можем закодировать файлы в строку и передавать их таким образом. 
В своём руководстве я использовал [сервис](https://8500.ru/file2base64/) для того, чтобы сгенерировать строку данного формата. Мне было необходимо предоставить несколько статичных данных для теста, поэтому я вставил полученные строки в вспомогательный класс и обращался к ним из кода. 
![](https://i9.wampi.ru/2020/04/18/58e5daeba01bc4df7.png)

Для генерации таких строк можно также воспользоваться встроенными средствами языка C#.

# Глава 5. Публикация сервера.
Для установки сервера в качестве weebhook требуется этот самый сервер загрузить в Интернет. Для этого можно воспользоваться сервисами по предоставлению услуг хостинга, vps или vds серверов.
Я воспользовался одним из популярных хостингов. 
+ [Реферальная ссылка](https://www.reg.ru?rlink=reflink-5211243)
+ [Обычная ссылка](https://www.reg.ru/).

Необходимо выбрать и оплатить услугу, которая поддерживает технологию ASP.Net.
Далее можно воспользоваться данной [инструкцией](https://www.reg.ru/support/hosting-i-servery/kak-razmestit-sayt-na-hostinge/kak-razvernut-sait-na-asp-net-s-pomoshiu-web-deploy) по публикации сервера на хостинг. 

# Возможные проблемы, с которыми вы можете столкнуться
+ Невозможно соединиться с сервером хостинга для публикации своего сервера. Решение - обратиться в техподдержку и попросить включить Web Deploy для вашей услуги
![](https://i9.wampi.ru/2020/04/18/68a4e2c6c1a134bd7.png)

+ **HTTP ERROR 500.0** [Решение](https://stackoverflow.com/questions/55731142/error-trying-to-host-an-asp-net-2-2-website-on-plesk-onyx-17-8-11-http-error-5)


# Ссылки
+ **Весь проект доступен на [GitHub](https://github.com/oxybes/ChatApiWA-InstructionsCSharp)**
+ **[API WhatsApp](chat-api.com)**
+ **[Документация по API](https://chat-api.com/ru/docs.html)**
+ **[JsonToC#](http://json2csharp.com/)**
+ **Хостинг [Реф](https://www.reg.ru?rlink=reflink-5211243) [Обычная](https://www.reg.ru/)**
