# Обзор возможностей разработки смарт-контрактов на Tact для TON

Tact – это высокоуровневый язык для смарт-контрактов блокчейна TON, ориентированный на эффективность и простоту разработки ￼. Он компилируется в код FunC (низкоуровневый язык TON) и избавляет разработчика от многих рутинных деталей: например, автоматически сериализует структуры данных и сообщения, тогда как в FunC эту логику нужно писать вручную ￼. Благодаря статической типизации и синтаксису, близкому к современным языкам, Tact упрощает создание сложных контрактов по сравнению с FunC ￼. Ниже рассмотрены ключевые возможности разработки на Tact для TON с примерами кода и пояснениями, включая преимущества Tact в каждом случае.

## Upgradable контракты (обновляемые контракты)

Обновляемые смарт-контракты позволяют менять код или данные уже развёрнутого контракта. В TON такая возможность существует, но используется с осторожностью из-за рисков безопасности и доверия ￼. Tact поддерживает механизмы апгрейда, предоставляя стандартные черты (traits) для обновления контракта, например, Upgradable и DelayedUpgradable.

Черта Upgradable (вместе с чертой Ownable для управления владельцем) добавляет обработку специального сообщения Upgrade с новым кодом и/или данными. В примере ниже определено сообщение Upgrade и черта Upgradable:

```
import "@stdlib/ownable";

// Сообщение для обновления контракта (код и/или данные)
message Upgrade {
    code: Cell? = null;  // новый код (optional)
    data: Cell? = null;  // новые данные (optional)
}

// Черта, реализующая механизм апгрейда с проверкой владельца
trait Upgradable with Ownable {
    owner: Address;            // адрес владельца, имеющего право на апгрейд
    _version: Int as uint32;   // версия контракта (увеличивается при каждом апгрейде)

    receive(msg: Upgrade) {
        let ctx = context();
        self.validateUpgrade(ctx, msg);  // проверка, что вызов от владельца
        self.upgrade(ctx, msg);         // выполнение обновления (смена кода/данных)
    }
}
```

Когда владелец отправляет внешнее сообщение Upgrade на адрес контракта, в receive сначала проверяется право (в validateUpgrade) и затем вызывается self.upgrade. Внутри этих вспомогательных методов происходит замена кода или данных: вызываются TVM-функции setCode и setData для применения нового кода/данных контракта ￼. После выполнения транзакции контракт будет работать уже с новым кодом. Переменная _version может использоваться для хранения текущей версии (например, для логирования или внешнего запроса версии).

Tact также предоставляет черту DelayedUpgradable для отложенного апгрейда (с таймлоком). В этом случае процесс проходит в два сообщения: сначала отправляется Upgrade с указанием задержки, контракт сохраняет обновление и время инициирования, а спустя указанное время владелец должен прислать подтверждение Confirm ￼. Если подтверждение придёт слишком рано, апгрейд не произойдёт. Ниже фрагмент реализации отложенного обновления (подтверждение вызывает setCode/setData при условии истечения таймлока):

```
receive(msg: Confirm) {
    require(now() >= self.initiatedAt + self.upgradeInfo.timeout,
            "Cannot confirm upgrade before timeout");
    if (self.upgradeInfo.code != null) {
        setCode(self.upgradeInfo.code!!);  // применить новый код (в конце транзакции)
    }
    if (self.upgradeInfo.data != null) {
        setData(self.upgradeInfo.data!!);  // применить новые данные немедленно
        stop(); // предотвратить повторный setData() в конце транзакции
    }
    _version += 1;
}
```

Как видно, Tact предоставляет готовые шаблоны для обновляемости контрактов, что упрощает реализацию. Преимущество Tact перед FunC здесь в том, что многие низкоуровневые детали уже учтены: достаточно подключить стандартную библиотеку и использовать готовую черту. В FunC же разработчику пришлось бы вручную писать логику проверки владельца, формирования нового stateInit и вызова set_code/set_data, тщательно отслеживая формат ячеек и остаток газа. Tact-интерфейсы инкапсулируют эту сложность. Тем не менее, разработчик должен использовать апгрейды осторожно – сообщества обычно ожидают неизменяемости смарт-контрактов, и возможность скрыто заменить код может подорвать доверие ￼ (например, открывая путь к мошенничеству rug pull). Рекомендуется добавлять таймлок и прозрачное голосование сообщества перед обновлением кода.

## Anti-whale механизмы (защита от «китов»)

Anti-whale (анти-китовые) механизмы направлены на защиту протокола или токена от слишком крупных участников («китов»), чтобы они не могли диспропорционально влиять на систему. Такие меры широко используются в DeFi – например, платформы могут вводить ограничения на размер транзакции, более высокие комиссии/налоги для крупных переводов, динамическое изменение цены при крупных ордерах, ограничения на владение определённым процентом от эмиссии и т.д. В криптопроектах эти «anti-whale provisions» стали стандартной практикой для предотвращения манипуляций и резких колебаний рынка ￼.

На уровне смарт-контракта TON реализовать anti-whale логику можно с помощью условий (require) и расчётов внутри функций приёма сообщений. В языке Tact это делается просто и понятно. Рассмотрим два распространённых подхода: лимит на транзакцию и динамическое ценообразование/налог.
	•	Лимит на транзакцию. Например, в контракте токена (Jetton) можно ограничить максимальное количество токенов, которое можно перевести или купить за одну транзакцию. В коде ниже показано, как отклонить перевод, превышающий заданный лимит:

```
const MAX_TRANSFER: Int = 100_000 * 1e9;  // лимит, напр. 100k токенов (в нанотокенах)

receive(msg: InternalTransfer) {
    require(msg.amount <= MAX_TRANSFER, "Transfer exceeds whale limit");
    ... 
    // продолжение обработки перевода (если условие прошло)
}
```

Здесь InternalTransfer – это стандартное сообщение стандарта Jetton (по TEP-74), с полем amount. Проверка require прервёт выполнение с ошибкой, если кто-то попытался перевести более MAX_TRANSFER токенов за раз. Таким образом, ни один «кит» не сможет одномоментно перебросить огромный объём токенов. Аналогично можно ограничить максимальную долю токенов, которой может владеть один адрес, проверяя баланс получателя до/после трансфера.

Динамические комиссии или цена. Другой подход – увеличивать «стоимость» операции для крупных объёмов. Например, можно ввести налог на крупных продавцов: если отправляемый объём превышает порог, часть токенов сгорает или идёт в пул вознаграждений. Или в контракте продажи токенов (ICO/IDO) можно сделать плавающую цену – чем больше токенов покупает пользователь, тем выше цена за токен (модель против скупки большого объёма по дешёвой цене). Ниже приведён упрощённый пример фрагмента контракта продажи, где реализована прогрессивная цена:

```
contract Sale {
    basePrice: Int = 1 ton;       // базовая цена за 1 токен (например, 1 TON)
    tokensSold: Int;             // уже продано токенов (для расчёта динамики)

    message Buy { amount: Int }  // сообщение покупки определённого кол-ва токенов

    receive(msg: Buy) {
        let currentPrice = basePrice + (self.tokensSold / 100_000) * 0.1 ton;
        ;; // цена растёт каждые 100k проданных токенов на 0.1 TON
        let cost = currentPrice * msg.amount;
        require(context().value >= cost, "Insufficient payment for purchase");
        self.tokensSold += msg.amount;
        // отправляем покупателю Jettons и сдачу, логируем продажу и т.д.
    }
}
```

В этом условном примере currentPrice увеличивается по мере того, как tokensSold растёт (имитация bonding curve: после каждых 100k продаж цена повышается). require гарантирует, что отправлено достаточно средств по новой цене. Такой механизм затрудняет одному участнику купить сразу очень большую долю токенов по низкой цене – крупная покупка мгновенно поднимет цену для этого объёма.

Anti-whale правила могут быть разнообразны: ограничение ежедневного объёма торгов для адреса, прогрессивный slippage (проскальзывание цены) на DEX при большой сделке, чёрные списки известных крупных кошельков и т.п. Реализация этих стратегий на Tact сводится к программированию соответствующих условий и математических формул, пользуясь богатым синтаксисом языка (операторы, арифметика с большими целыми, встроенные функции). Преимущество Tact – в читабельности и скорости разработки таких правил: проверки на превышение лимитов или вычисление динамической комиссии в Tact выглядят декларативно, почти как псевдокод, тогда как в FunC те же проверки требуют писать более развернутый код с явной работой со слайсами и ячейками. Это снижает вероятность ошибки и облегчает аудит контракта на наличие «лазеек» для китов. В результате, с помощью Tact разработчики легко добавляют в свои Jetton-контракты anti-whale механизмы, делая токеномику более справедливой ￼ и защищённой от манипуляций.

## Фарминг и стейкинг (вознаграждения за депозиты и блокировка токенов)

Стейкинг и фарминг – ключевые компоненты DeFi на TON, позволяющие пользователям получать пассивный доход за участие в протоколе. Обычно под стейкингом понимают блокировку токенов на определённое время для поддержки работы сети или протокола, за что начисляются вознаграждения ￼. На блокчейне TON классический пример – стейкинг Toncoin для валидаторов или делегирование монет валидаторским пулам. Фарминг (или yield farming, liquidity mining) – более широкое понятие, означающее предоставление своих активов (например, ликвидности в пул DEX либо токенов в протокол кредитования) в обмен на награды ￼. Пользователи фармят, например, вкладывая Jetton-токены в пул ликвидности и получая доход от комиссий DEX + дополнительные вознаграждения от протокола.

На языке Tact можно реализовывать смарт-контракты для стейкинга и фарминга, которые будут управлять депозитами пользователей, начислять награды и позволять вывод средств. Рассмотрим упрощённый пример контракта стейкинга Jetton-токенов. Допустим, у нас есть токен (Jetton), и мы хотим создать контракт, куда пользователи смогут отправлять эти токены для фарминга – контракт будет их удерживать, а затем распределять вознаграждение (например, другие токены или увеличивать баланс). Такой контракт обычно состоит из: функции приёма депозита, хранения информации о вкладах, функции для вывода (unstake) и, возможно, периодической выдачи наград.

Пример: контракт принимает уведомления о переводе определённого Jetton (Jetton Transfer Notification) и учитывает баланс каждого вкладчика:

```
import "@stdlib/ownable";  // пусть контракт управляется админом (для настройки, выемки наград и т.д.)

message TokenDeposit { amount: Int }  // кастомное сообщение, если нужно инициировать stake иначе

// Структура записи о стейке пользователя
struct StakeInfo {
    amount: Int;       // количество застейканных токенов
    timestamp: Int;    // время депозита (для расчёта вознаграждений)
}

contract FarmingContract with Ownable {
    totalStaked: Int;
    stakes: map<Address, StakeInfo>;   // мэп адрес -> информация о стейке

    // Получение уведомления о переводе Jetton (по стандарту Jetton wallet вызывает receive для уведомления)
    receive(jetton: JettonTransferNotification) {
        ;; // Проверим, что это нужный Jetton (по адресу отправителя или через payload)
        require(context().value >= 0.1 ton, "Not enough gas attached");
        let senderAddr = jetton.sender;
        let amount = jetton.amount;
        // Обновляем общий застейканный объем
        self.totalStaked += amount;
        // Добавляем или обновляем запись о стейке данного пользователя
        let prev = self.stakes.get(senderAddr);
        if (prev == null) {
            // новый стейкер
            self.stakes.set(senderAddr, StakeInfo{ amount: amount, timestamp: now() });
        } else {
            // уже был стейк – складываем
            let newAmount = prev.amount + amount;
            self.stakes.set(senderAddr, StakeInfo{ amount: newAmount, timestamp: now() });
        }
        ;; // Опционально: выдавать мгновенный reward токеном, эмитить событие, др.
    }

    // Сообщение на вывод стейка
    message Unstake {}

    receive(msg: Unstake) {
        let user = context().sender;
        let stakeInfo = self.stakes.get(user);
        require(stakeInfo != null && stakeInfo.amount > 0, "No stake to withdraw");
        // Расчет вознаграждений (например, просто время * ставка * коэффициент)
        let stakedAmt = stakeInfo.amount;
        let duration = now() - stakeInfo.timestamp;
        let reward = (stakedAmt * duration) / 1000000;  // условный расчет награды
        // Обновляем данные
        self.stakes.delete(user);
        self.totalStaked -= stakedAmt;
        // Возвращаем пользователю его токены + вознаграждение (предположим вознаграждение в TON)
        // Отправляем обратно Jetton токены (через внутреннее сообщение Jetton-wallet) и награду TON
        send(SendParameters{
            value: reward,            // награда в TON
            to: user, 
            bounce: false
        });
        ;; // Для возврата Jetton токенов потребовался бы вызов Jetton wallet контракта, опущено для простоты
    }
}
```

В этом коде контракт хранит в stakes баланс стейка каждого пользователя и метку времени. Функция receive(jetton: JettonTransferNotification) срабатывает, когда на адрес контракта приходит перевод определённого Jetton (поля sender, amount и forwardPayload позволяют идентифицировать от кого и сколько токенов пришло ￼). Мы требуем прикрепить немного TON (0.1 ton) для покрытия газа операции. Затем увеличиваем счётчик totalStaked и записываем/обновляем информацию о стейке отправителя.

Сообщение Unstake позволяет вывести токены. Контракт проверяет, есть ли у отправителя застейканные токены, вычисляет reward (здесь для иллюстрации пропорционально времени удержания и сумме; на практике формула может быть более сложной). Затем удаляет запись стейка и возвращает пользователю его вклады. Возврат Jetton-токенов обычно реализуется вызовом функции внутреннего перевода из Jetton-wallet контракта, но здесь для краткости представлено как концептуально выполненное действие. Награды могут выдаваться в TON (как показано) или в других токенах/Jetton – в зависимости от модели фарминга. Например, контракт может начислять собственный токен проекта за время стейкинга.

Фарминг часто строится поверх стейкинга: пользователь стейкает LP-токены (доля ликвидности DEX) и получает вознаграждение от протокола. Отличие лишь в том, что вместо одного токена стейкается пара токенов (через LP-токен), и награда может распределяться из пула, управляемого контрактом.

Преимущества использования Tact для таких контрактов очевидны. Язык предоставляет удобные структуры данных (map, struct), позволяющие хранить информацию о многих участниках без сложного манипулирования памятью. В FunC, чтобы достичь того же, пришлось бы работать с ассоциативными массивами на основе хеш-ключей вручную и парсить/сериализовать структуры – в Tact всё это автоматически обрабатывается компилятором ￼. Код на Tact значительно короче и яснее. Например, одна строка self.stakes.get(senderAddr) распаковывает запись о стейке, тогда как на FunC потребовалось бы пройти по данным в ячейке или использовать встроенные ассемблерные примитивы для работы с хранилищем. Это снижает вероятность багов при расчёте наград и распределении средств. Кроме того, Tact позволяет легко интегрироваться со стандартом Jetton: определение сообщений для взаимодействия (как JettonTransferNotification) и их обработка выполняются через понятные декларации, а не ручное разбиение бинарных payload’ов.

Таким образом, с помощью Tact можно быстро реализовать надёжные контракты для стейкинга и фарминга на TON. Пользователи блокируют свои токены, контракт учитывает их доли и автоматически (по вызову или периодически) распределяет награды – всё прозрачно и без участия централизованных посредников, что и является сутью DeFi.

## DAO (децентрализованные автономные организации) – голосование и управление средствами

Децентрализованная автономная организация (DAO) – это смарт-контракт(ы), которые позволяют сообществу принимать коллективные решения посредством голосования. DAO обычно используют токены управления: вес голоса участника пропорционален количеству токенов (или другим критериям). Идея в том, что протокол управляется самими пользователями, а не централизованно. Реализуется это через набор функций: создание предложений, голосование и исполнение решений – например, перевод средств казны или изменение параметров протокола, одобренные голосованием ￼. В Эфириуме есть готовые фреймворки (OpenZeppelin Governor и др.) ￼; в TON разработчик может написать подобную логику на Tact.

Рассмотрим упрощённый пример DAO-контракта на Tact, который позволяет участникам вносить предложения и голосовать за них. Чтобы не усложнять пример интеграцией с токеном, предположим, что в контракте заложен список участников или любой может проголосовать один раз (модель «один адрес – один голос») – на практике можно учитывать баланс Jetton-токена для взвешенных голосов, считывая его через get-запросы или требуя депозита токенов в DAO-контракт.

Наш DAO-контракт будет хранить предложения в структуре, содержащей счетчики голосов «за»/«против», флаг выполнения и, к примеру, информацию о транзакции, которую нужно выполнить при одобрении (назначение средств).

Пример структуры и сообщений DAO:

```
struct Proposal {
    id: Int;
    yesVotes: Int;
    noVotes: Int;
    endTime: Int;
    executed: Bool;
    target: Address;   // адрес, куда отправить средства при выполнении
    amount: Int;       // сумма TON для отправки при выполнении
}

message CreateProposal {
    id: Int;
    duration: Int;     // период голосования (в секундах)
    target: Address;
    amount: Int;
}

message Vote {
    proposalId: Int;
    support: Bool;     // true = голос "за", false = "против"
}

message Execute {
    proposalId: Int;
}

Основная логика DAO-контракта:

contract SimpleDAO {
    proposals: map<Int, Proposal>;

    // Создание нового предложения
    receive(msg: CreateProposal) {
        require(self.proposals.get(msg.id) == null, "Proposal ID already exists");
        let prop = Proposal{
            id: msg.id,
            yesVotes: 0,
            noVotes: 0,
            endTime: now() + msg.duration,
            executed: false,
            target: msg.target,
            amount: msg.amount
        };
        self.proposals.set(msg.id, prop);
    }

    // Голосование по предложению
    receive(msg: Vote) {
        let prop = self.proposals.get(msg.proposalId);
        require(prop != null, "Proposal not found");
        require(now() < prop.endTime, "Voting period ended");
        // (Опционально: проверить, что голосующий не голосовал ранее, или реализовать учет веса)
        if (msg.support) {
            prop.yesVotes += 1;
        } else {
            prop.noVotes += 1;
        }
        self.proposals.set(prop.id, prop);
    }

    // Исполнение предложения (если набрано большинство голосов "за")
    receive(msg: Execute) {
        let prop = self.proposals.get(msg.proposalId);
        require(prop != null, "Proposal not found");
        require(now() >= prop.endTime, "Voting still in progress");
        require(!prop.executed, "Already executed");
        require(prop.yesVotes > prop.noVotes, "Proposal not approved");
        // Помечаем выполненным
        prop.executed = true;
        self.proposals.set(prop.id, prop);
        // Выполняем заложенное действие – перевод средств
        if (prop.amount > 0) {
            send(SendParameters{
                value: prop.amount,
                to: prop.target,
                bounce: false
            });
        }
    }
}
```

В этом смарт-контракте DAO есть хранилище предложений proposals (ассоциативный массив по id). Когда участник отправляет транзакцию с сообщением CreateProposal, создаётся новая запись: устанавливаются счётчики голосов в 0, вычисляется endTime как текущий время + длительность голосования, сохраняется адрес и сумма для будущей выплаты. Голосование (Vote) увеличивает либо yesVotes, либо noVotes. (Заметим, что тут не контролируется, чтобы один адрес не голосовал несколько раз – в реальном контракте это важно учесть, например, через хранение флага голосования в мэпе voters или требование вложить токены голосования). Наконец, сообщение Execute проверяет условия окончания голосования: время истекло, предложение ещё не выполнялось, и голосов “за” больше, чем “против”. Если всё верно – контракт отправляет на заранее заданный target указанную сумму amount из средств, находящихся на балансе DAO-контракта (то есть контракт должен владеть некими средствами – казной DAO).

Такой простой DAO-конракт демонстрирует принцип: смарт-контракт автоматически исполняет решение, принятое большинством голосов, без вмешательства третьих лиц. Средства DAO находятся под управлением контракта и расходуются только согласно результатам голосования, что обеспечивает прозрачность и автономность организации.

Конечно, в боевых DAO обычно есть более сложные механики: взвешенные голоса по токенам (например, 1 токен = 1 голос), кворум (минимальное число голосов для легитимности), возможность делегирования голоса другим, криминальные пороги (например, нельзя тратить более определённой суммы без супербольшинства), время удержания токена для права голоса (так называемый vesting голоса) и т.д. Но всё это можно реализовать, дополняя контракт новыми полями и проверками. В Tact за счёт поддержки сложных структур данных и логики программирование таких функций значительно облегчается. Например, можно легко интегрировать Jetton-токен как токен управления: Jetton Master контракт может предоставлять через get-метод баланс токенов у каждого адреса, и DAO-контракт перед учётом голоса будет запрашивать у Jetton Master баланс голосующего адреса, чтобы учесть вес голоса пропорционально токенам. Либо, как упомянуто, заставить участников stake (депонировать) свои токены управления в DAO-контракт, тогда stakes.get(voter) даст вес голоса прямо, без внешних запросов.

Преимущества Tact перед FunC особенно заметны в подобных сложных сценариях. В FunC реализация DAO потребовала бы вручную определять бинарные форматы для сообщений предложений и голосований, парсить их внутри recv_external, вести сериализацию/десериализацию структуры предложения при каждом обновлении (увеличении счетчика голосов), т.е. много шаблонного кода. Tact же автоматизирует сериализацию структур: определив struct Proposal и поместив его в map, мы можем просто вызывать self.proposals.set(...) с обычной записью, а компилятор позаботится о представлении её внутри persistent-хранилища контракта. Это подтверждает и официальная документация: в Tact сериализация структур и сообщений происходит автоматически, тогда как в FunC разработчик вынужден определять её вручную ￼. Кроме того, Tact позволяет разбивать логику по функциям (методам) обработки сообщений (receive) – код естественно структурирован по типам входящих действий, вместо одного громоздкого recv_internal с большим switch по операциям. Всё это ускоряет разработку и уменьшает вероятность ошибок, критичных для безопасности DAO (например, ошибки, позволяющей выполнить предложение без достаточного числа голосов).

Таким образом, Tact предоставляет удобный инструментарий для построения DAO на TON: от простейших голосовалок до сложных многокон-трактных систем с токенами управления. Уже существуют проекты (например, ton.vote) реализующие on-chain голосования в TON ￼ ￼. С Tact разработчики DAO получают преимущества быстрого прототипирования и встроенных средств безопасности (таких как Ownable, проверка подписей и пр.), что позволяет сосредоточиться на логике управления сообществом, не отвлекаясь на низкоуровневые детали.

## Multisig_v2 – мультиподписи для безопасности транзакций

Multisig (мультиподпись) – это смарт-контракт-кошелёк, для отправки транзакции из которого требуется подтверждение несколькими ключами (подписями) из заранее определённого набора. На TON реализовано несколько версий мультисиг-кошельков, одна из популярных – SafeMultisigWallet v2. В такой кошелёк могут быть добавлены M совладельцев (адресов), и для проведения транзакции необходимо как минимум N подписей (N ≤ M). Например, мультисиг 2-из-3: 3 владельца, нужно как минимум 2 согласных, чтобы перевод состоялся. Это существенно повышает безопасность хранения средств: компрометация одного ключа не позволит украсть средства без участия других владельцев.

Мультисиг-кошелёк обычно работает так: один из владельцев формирует запрос транзакции (куда, сколько отправить, с какими параметрами) – это записывается как неподтверждённый ордер. Затем другие владельцы отправляют подтверждения (подписывают ордер). Когда набирается достаточное число подтверждений (N), кошелёк автоматически выполняет заложенную транзакцию. Если необходимое число подписей не собрано, транзакция не происходит.

На Tact можно реализовать мультисиг-контракт как единый контракт с хранением структуры ордеров и счетчиков подтверждений, либо, как сделано в официальном примере Tact, через специальный подконтракт для каждого ордера. Рассмотрим обе идеи.

1. Простая реализация одним контрактом. Внутри контракта храним список владельцев (например, map<Address, Bool> или map<Address, Int> если у каждого разный «вес») и порог requiredSigns – сколько подписей нужно. Для простоты предположим равные голоса. В контракте будет храниться структура PendingTx (неподтверждённая транзакция) с полями: адрес назначения, сумма, собранные подтверждения и список уже подписавших. Ниже приведён псевдокод такой реализации:

```
struct PendingTx {
    to: Address;
    amount: Int;
    approvals: Int;
    approvedBy: map<Address, Bool>;
    executed: Bool;
}

contract MultisigWallet {
    owners: map<Address, Bool>;   // список владельцев
    requiredSigns: Int;
    pending: map<Int, PendingTx>; // множество активных ордеров (по id)

    // Сообщение для предложения новой транзакции
    message Propose { id: Int; to: Address; amount: Int }

    // Сообщение для подписания (подтверждения) предложенной транзакции
    message Approve { id: Int }

    init(ownersList: map<Address, Bool>, reqSigns: Int) {
        self.owners = ownersList;
        self.requiredSigns = reqSigns;
    }

    // Инициация нового ордера
    receive(msg: Propose) {
        require(self.owners.contains(sender()), "Not an owner");
        require(self.pending.get(msg.id) == null, "ID already used");
        // Создаем новый PendingTx
        let newOrder = PendingTx{
            to: msg.to,
            amount: msg.amount,
            approvals: 0,
            approvedBy: map<Address,Bool>{},
            executed: false
        };
        // Если желаем, можно сразу засчитать подпись инициатора:
        // newOrder.approvedBy.set(sender(), true);
        // newOrder.approvals += 1;
        self.pending.set(msg.id, newOrder);
    }

    // Подписание ордера
    receive(msg: Approve) {
        require(self.owners.contains(sender()), "Not an owner");
        let order = self.pending.get(msg.id);
        require(order != null, "Order not found");
        require(!order.executed, "Order already executed");
        require(order.approvedBy.get(sender()) == null, "Already approved by this owner");
        // Регистрируем подпись
        order.approvedBy.set(sender(), true);
        order.approvals += 1;
        // Если достигнут порог - выполняем транзакцию
        if (order.approvals >= self.requiredSigns) {
            order.executed = true;
            send(SendParameters{
                value: order.amount,
                to: order.to,
                bounce: false
            });
        }
        self.pending.set(msg.id, order);
    }
}
```

В этой реализации, когда достаточно владельцев вызвали Approve для конкретного ордера, контракт сразу отправляет указанные средства (order.amount в адрес order.to). Каждый владелец проверяется по списку owners. Мы следим, чтобы один владелец не засчитал две подписи (approvedBy хранит отметки). В более полном решении стоило бы предусмотреть срок действия ордера (expiration time) и возможность его удаления, а также seqno/nonce для предотвращения повтора старых сообщений. Тем не менее, принцип «N-из-M подписей» продемонстрирован.

2. Реализация через вспомогательный контракт на каждый ордер. Tact позволяет динамически создавать смарт-контракты прямо из контракта, используя функцию initOf для генерации StateInit нового контракта и отправки сообщения для его создания ￼ ￼. Официальный пример Multisig из репозитория Tact как раз применяет такую модель: основной контракт Multisig при получении Request (аналог Propose) генерирует дочерний контракт MultisigSigner, передавая ему список владельцев, требуемый порог и детали запроса. Этот подп-contract собирает подтверждения (в его коде есть обработка сообщений "YES" от владельцев, увеличивающая счётчик) ￼ ￼. Когда подп-контракт набирает достаточно голосов, он шлёт основному контракту сообщение Signed с деталями запроса ￼ ￼. После этого главный контракт выполняет транзакцию аналогично примеру выше ￼. Такой подход более модульный: каждый активный запрос – отдельный контракт, изолированно ожидающий голоса. После исполнения эти временные контракты обычно саморазрушаются или остаются помеченными как completed.

Использование отдельного контракта на запрос даёт гибкость (можно параллельно обрабатывать несколько запросов), но и больше накладных расходов (создание нескольких контрактов, хранение их в блокчейне до завершения). В рамках Tact реализация упрощается: метод initOf получает на вход имя контракта (определённого в том же файле), и параметры для его конструктора. Например, в коде вышеизложенного официального примера есть строчки:

```
let opInit: StateInit = initOf MultisigSigner(myAddress(), self.members, self.requiredWeight, msg);
let opAddress: Address = contractAddress(opInit);
send(SendParameters{
    value: 0,
    to: opAddress,
    bounce: true,
    code: opInit.code,
    data: opInit.data
});
```

Этот фрагмент генерирует StateInit для нового контракта MultisigSigner (передавая в него адрес «мастера» и параметры голосования), вычисляет его будущий адрес и отправляет сообщение самому себе с прикреплёнными code и data нового контракта, что инициирует его развертывание ￼ ￼. В FunC подобное потребовало бы вручную составлять ячейку stateInit, вычислять адрес через хеш, и формировать внутреннее сообщение с отдельными полями кода/данных – Tact делает это автоматически и безопасно.

Независимо от подхода, Multisig_v2 на TON обеспечивает надёжную многостороннюю схему управления средствами, устраняя единственную точку отказа (один ключ). Такие кошельки применяются для командной работы с казной проектов, для хранения резервов на биржах, совместного управления активами и т.д. Контракт мультисиг N-из-M гарантирует, что без согласия достаточного числа совладельцев транзакция не выполнится ￼. В то же время, мультисиг остаётся достаточно гибким: можно заложить несколько действий в один ордер (например, отправку токенов и изменение списка владельцев), поскольку по сути ордер – это набор произвольных внутренних сообщений от имени кошелька ￼.



Источники:
	1.	Tact Documentation – Contract upgrades ￼ ￼
	2.	Tact Documentation – Code and data upgrades (cookbook) ￼
	3.	Cointelegraph – Anti-whale provisions in DeFi platforms ￼
	4.	STON.fi Blog – Yield farming and staking definitions ￼ ￼
	5.	Tact by Example (GitHub) – Multisig wallet implementation ￼ ￼
	6.	TON Blockchain repository – Multisig wallet v2 description ￼ ￼
	7.	CertiK – Secure Smart Contract Programming in Tact ￼
	8.	Tact Documentation – Serialization vs FunC ￼
	9.	Smart Contract Tips – DAO Governor Contracts overview ￼