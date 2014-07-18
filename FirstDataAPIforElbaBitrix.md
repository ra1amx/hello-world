От:	kadyrov@skbkontur.ru
Кому:	Ivan Eremin <ivan.eremin@mail.ru>
Дата:	Птн, 11 Июл 2014 12:34
Тема:	апи для битрикс24



День добрый 

Ниже описание API для интеграции с битрикс24 

С уважением, 
Кадыров Андрей 






Доступны два хэндлера - CreateBill и GetChanges. Первый создает счет в Эльбе, второй позволяет получить 
изменения, которые происходили в системе с созданными через CreateBill счетами. 
UI редактирования счетов, созданных через CrateBill в нашей системе остается стандартным. 
В том числе, пользователь может удалить такой счет. 

В http-хидерах передается логин/пароль от существующей учетной записи в Эльбе. 
В учетной записи должен быть куплен тариф, позволяющий создавать счета. 
В теле запроса - json с данными по создаваемому счету. Если переданных контрагента или товара у нас нет, 
мы их создаем. Сопоставление для контрагента - по имени, инн/кпп, номеру банк. счета. 
Для товаров - по имени, возможно по артикулу (если он есть). 
BankAccount не обязателен, но если передан, требуется указать банк (имя, бик). 
Если счет успешно создан, http status code = 200, в теле возвращается id (guid) созданного счета. 

Неверный логин/пароль - Forbidden 

Абонент в Эльбе не может являться обслуживающей бухгалтерией и у него не должна быть 
включена СМС авторизация, иначе - Forbidden. 

При прочих нехороших ситуациях возвращается BadRequest (не заполнены обязательные поля, абонент не оплачен). В body - описание ошибки. 

В случае непредвиденных ошибок возвращается BadRequest, в теле - Message + номер ошибки. 
По номеру ошибки мы в своих логах сможем найти детали проблемы. 

POST 
https://elba.kontur.ru/PublicInterface...eBill.ashx 
headers: 
X-Login: urlencoded(login) 
X-Password: urlencoded(password) 
body: 
{ 
//required, length <= 100 
"Number": "аб324", 

//ISODate, required 
"Date": "2014-02-07T00:00:00.000Z", 

//счет с ндс 
"WithNds": true, 

//цена и сумма в фактурной части включают НДС 
"SumsWithNds": true 

//not required, ntext 
"Comment": "test comment", 

//not required, счет организации 
"BankAccount": { 
"AccountNumber": "40802810363030003088", 
"Bank": { 
"Name": "ОАО \"АЛЬФА-БАНК\"", 
"Bik": "044525593", 
"City": "МОСКВА", 
"CityType": "Г.", 
"CorrAccount": "30101810200000000593" 
} 
}, 
"Contractor": { 
//required, length <= 2000 
"Name": "ИП Иванов Иван Иванович", 
//not required 
"Inn": "6660013279", 
//not required 
"Kpp": "666100042" 
}, 
"Items": [ 
{ 
//not required, length < 1000 
"ProductName": "стул", 
//not required, length < 1000 
"UnitName": "шт", 
"Quantity": 10.0, 
"Price": 123.06, 
"PriceWithoutNds": 104.29, 
"Sum": 1230.6, 
"NdsRate": 3 
}, 
{ 
"ProductName": "кабель", 
"UnitName": "м", 
"Quantity": 10.0, 
"Price": 100.0, 
"PriceWithoutNds": 100.0, 
"Sum": 1000.0, 
"NdsRate": 0 
}, 
{ 
"ProductName": "апельсины", 
"UnitName": "кг", 
"Quantity": 10.0, 
"Price": 189.56, 
"PriceWithoutNds": 160.64, 
"Sum": 1895.60, 
"NdsRate": 2 
} 
] 
} 
//NdsRate: 
// WithoutNds = 0, 
// Nds0 = 1, 
// Nds10 = 2, 
// Nds18 = 3 


================== 

GetChangds принимает номер ревизии fromRevision: long, и возвращает описания всех созданных через CreateBill счетов 
с ревизиями, большими fromRevision в виде массива 
changes: [] { 
id: guid, 
status: status 
revision: long 
} 
status -> notPaid, Paid, PartiallyPaid, Rejected 

При первом вызове в качестве fromRevision следует передавать 0, в остальных - max(changes.revision). 
Каждое изменение счета в нашей системе обладает своей ревизией, поэтому GetChanges будет возвращать 
все изменения по счетам, в том числе не связанные с состоянием оплаченности. Клиент может на своей 
стороне проверить, что статус счета действительно изменился. 

GET 
https://elba.kontur.ru/PublicInterface...evision=42 
headers: 
X-Login: urlencoded(login) 
X-Password: urlencoded(password) 

ответ 
body: 
[ 
{ 
"Id": "7b9c0329-c640-4c4e-868a-900df8f6e5c5", 
"Revision": 123, 
"Status": 1 
}, 
{ 
"Id": "53df594a-eed4-44ba-bde1-f1d26b42c3b3", 
"Revision": 89, 
"Status": 0 
} 
] 
//Status: 
// NotPaid = 0 
// Paid = 1 
// PartiallyPaid = 2 
// Rejected = 3 

Тестовый стенд: 
https://dontpedro.skbkontur.ru/ib.mast...eBill.ashx, 
https://dontpedro.skbkontur.ru/ib.mast...anges.ashx. 

Для создания абонентов/входа в существующих: 
https://dontpedro.skbkontur.ru/IB.Mast...Login.aspx 
https://dontpedro.skbkontur.ru/IB.Mast...ister.aspx. 
Чтобы сделать абонента оплаченным следует просто нажать "оплатить"/пластиковой картой/... - там фэйковая оплата. 
