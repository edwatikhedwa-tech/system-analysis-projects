# Тикет на разработку — фильтрация по эксклюзивности, Otium

Постановка задачи разработчику: доработка SOAP-метода стримингового сервиса и исправление XML-схем запроса и ответа.

---

## 1. Тикет

| Поле | Содержание |
| :--- | :--- |
| **Название** | Фильтрация списка фильмов и сериалов по эксклюзивности |
| **Цель** | Обеспечить быстрый поиск и отображение контента, доступного исключительно на платформе Otium, чтобы подчеркнуть уникальность подписки и повысить вовлечённость пользователей |
| **Требование** | Пользователь должен иметь возможность отфильтровать общий список фильмов и сериалов так, чтобы отображались только позиции, помеченные как эксклюзивные для Otium |

### Use Case

1. В разделе «Библиотека» пользователь выбирает чек-бокс «представлены эксклюзивно в Otium»
2. Система применяет фильтр и отображает только те фильмы и сериалы, у которых в данных стоит признак эксклюзивности
3. Если чек-бокс снят — показывается полный каталог без ограничения по эксклюзивности
4. Фильтр должен корректно работать в сочетании с другими фильтрами (жанр, год, рейтинг), если они выбраны одновременно

### Описание изменений

Доработать метод `{{WebServer}}/content/list` (SOAP):

| Что | Где | Как |
| :--- | :--- | :--- |
| Добавить элемент `exclusiveOnly` | тело запроса | `xs:boolean`, optional |
| Добавить элемент `exclusive` | тело ответа, в каждый объект `content` | `xs:boolean` |
| Добавить обработку `exclusiveOnly` | логика метода | `true` — из полного списка от `FilmsServer` и `SeriesServer` отобрать только записи с `exclusive = true`; `false` или параметр отсутствует — вернуть полный список без фильтрации |

### Контекст

В ответах `{{FilmsServer}}/films/list` и `{{SeriesServer}}/series/list` элемент `exclusive` **уже есть**. Новых источников данных задача не требует — нужна только фильтрация на стороне `WebServer` и проброс признака в ответ.

---

## 2. Схема запроса `GetContentListRequest.xsd`

### Найденные ошибки

| Строка | Ошибка |
| :--- | :--- |
| `<xs:element name="GetContentList">]` | Лишняя закрывающая скобка `]` — синтаксическая ошибка |
| `<xs:sequence>` | Отсутствует `<xs:complexType>`: `sequence` находился непосредственно внутри `element`, что нарушает правила XSD и делает схему невалидной |
| `<xs:element name="genreValue" />` | Не указан `type` |
| — | Отсутствует новый параметр `exclusiveOnly` |

### Исправленная схема

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema">
  <xs:element name="GetContentList">
    <xs:complexType>
      <xs:sequence>
        <xs:element name="contentType" type="xs:string"/>
        <xs:element name="genreValue" type="xs:string"/>
        <xs:element name="exclusiveOnly" type="xs:boolean" minOccurs="0"/>
      </xs:sequence>
    </xs:complexType>
  </xs:element>
</xs:schema>
```

**Что изменено:** убрана лишняя `]`; добавлен обязательный по правилам XSD `<xs:complexType>`; проставлен `type` у `genreValue`; добавлен `exclusiveOnly` с `minOccurs="0"` — параметр необязательный.

**Решение по значению по умолчанию.** `default` в схеме не задаётся: поведение при отсутствующем параметре описано в тикете как «вернуть полный список без фильтрации» и реализуется на стороне обработки. Значение по умолчанию, продублированное в схеме, — второй источник правды, который начнёт расходиться с логикой при первом же изменении требований.

---

## 3. Схема ответа `GetContentListResponse.xsd`

### Найденные ошибки

| Ошибка | Пояснение |
| :--- | :--- |
| `<xs:element name="content">` без `maxOccurs="unbounded"` | Список контента мог содержать только один элемент — верхняя граница количества не снята |
| `<xs:element name="contentId" />` | Отсутствует `type` |
| `<xs:element name="genreValue"/ type="xs:string" >` | Слэш стоит перед атрибутом `type` — некорректный синтаксис |
| `type="xs:string”` у `duration` | Закрывающая кавычка другого типа (типографская вместо прямой) |
| `<xs:element name=”” >` и `<xs:element name="" type="" />` | Пустые атрибуты `name` |
| Блоки `team` и `cast` | Не хватает закрывающих `</xs:sequence>`, `</xs:complexType>`, `</xs:element>` |
| — | Отсутствует элемент `exclusive` |

### Исправленная схема

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           xmlns:soap="http://www.w3.org/2003/05/soap-envelope/"
           soap:encodingStyle="http://www.w3.org/2003/05/soap-encoding">
  <xs:element name="GetContentListResponse">
    <xs:complexType>
      <xs:sequence>
        <xs:element name="contentList">
          <xs:complexType>
            <xs:sequence>
              <xs:element name="content" maxOccurs="unbounded">
                <xs:complexType>
                  <xs:sequence>
                    <xs:element name="contentType" type="xs:string"/>
                    <xs:element name="contentId" type="xs:integer"/>
                    <xs:element name="title" type="xs:string"/>
                    <xs:element name="description" type="xs:string"/>
                    <xs:element name="imageUrl" type="xs:string"/>
                    <xs:element name="previewUrl" type="xs:string"/>
                    <xs:element name="recordUrl" type="xs:string"/>
                    <xs:element name="genre">
                      <xs:complexType>
                        <xs:sequence>
                          <xs:element name="genreValue" type="xs:string"/>
                        </xs:sequence>
                      </xs:complexType>
                    </xs:element>
                    <xs:element name="recommended" type="xs:boolean"/>
                    <xs:element name="exclusive" type="xs:boolean"/>
                    <xs:element name="details">
                      <xs:complexType>
                        <xs:sequence>
                          <xs:element name="yearOfIssue" type="xs:string"/>
                          <xs:element name="duration" type="xs:string"/>
                          <xs:element name="country">
                            <xs:complexType>
                              <xs:sequence>
                                <xs:element name="countryValue" type="xs:string" maxOccurs="5"/>
                              </xs:sequence>
                            </xs:complexType>
                          </xs:element>
                          <xs:element name="ageRate" type="xs:string"/>
                        </xs:sequence>
                      </xs:complexType>
                    </xs:element>
                    <xs:element name="team">
                      <xs:complexType>
                        <xs:sequence>
                          <xs:element name="cast">
                            <xs:complexType>
                              <xs:sequence>
                                <xs:element name="castValue" type="xs:string"/>
                              </xs:sequence>
                            </xs:complexType>
                          </xs:element>
                          <xs:element name="dubbingTeam" type="xs:string"/>
                        </xs:sequence>
                      </xs:complexType>
                    </xs:element>
                    <xs:element name="rating" type="xs:decimal"/>
                  </xs:sequence>
                </xs:complexType>
              </xs:element>
            </xs:sequence>
          </xs:complexType>
        </xs:element>
      </xs:sequence>
    </xs:complexType>
  </xs:element>
</xs:schema>
```

### Сводка изменений

| Изменение | Тип |
| :--- | :--- |
| `content` получил `maxOccurs="unbounded"` | исправление |
| `contentId` — тип `xs:integer` | исправление |
| `genreValue`, `ageRate` — проставлен `type` | исправление |
| `duration` — исправлена кавычка | исправление |
| Пустой элемент переименован в `country` | исправление |
| Пустой элемент в `team` заменён на `dubbingTeam` | исправление |
| Восстановлены закрывающие теги в `team` и `cast` | исправление |
| Добавлен `exclusive` типа `xs:boolean` в объект `content` | **новое требование** |

---

## Принцип, по которому вносились правки

Исправлялись только те синтаксические ошибки, которые препятствовали валидации схемы: неправильные кавычки, пустые атрибуты `name`, некорректное расположение слэшей, недостающие закрывающие теги. По существу задачи в контракт добавлено **одно поле** — `exclusive` в объекте `content` и `exclusiveOnly` в запросе.

Разделение важно: правка контракта интеграции — рискованная операция, каждое лишнее изменение придётся согласовывать с потребителями API. Поэтому в тикете явно разграничены «починил, чтобы схема стала валидной» и «изменил, потому что этого требует задача».
