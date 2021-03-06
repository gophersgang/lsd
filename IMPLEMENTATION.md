# Детали реализации демона LSD
## Схема работы сервера
Сервер принимает по обычному GPB запросы от клиентов на то, чтобы записать новые смещения (request_new_events). В событии содержатся имя категории, inode файла, смещение в этом файле и массив строк, которые нужно записать. В ответ присылается inode и offset из оригинального сообщения — это выступает в роли acknowledge для клиента о том, что соответствующая порция данных успешно доставлена и записана в файл, чтобы клиент мог записать себе эту информацию в offsets_db.

На каждую категорию стартует по отдельной горутине, которая и занимается записью файлов в указанную директорию соответствующей категории. Операции с I/O идут не более, чем в MAX_IO_THREADS=16 потоков (в данный момент жестко зашито в коде), чтобы не создавать излишнюю нагрузку на сервер и не плодить треды ОС неконтролируемым образом, поскольку системные вызовы для работы с файлами являются блокирующими операциями.

В файлы запись осуществляет блоками не более 64 Кб.

## Схема работы клиента
Клиент с помощью ОС-специфичного механизма для слежения за изменениями в ФС (то есть, kqueue на Mac OS X/FreeBSD и inotify на Linux) следит за директорией source_dir из конфига, а также за её поддиректориями (это делается нерекурсивно — вложенные поддиректории не отслеживаются).

Потоки изменений доставляются в каналы для горутин, на каждую категорию создается по 2 горутины — одна горутина для непосредственного чтения событий с диска, и одна горутина для буферизации событий, которые относятся к соответствующей категории (необходимо для того, чтобы не «спамить» событиями в основную горутину, а также чтобы не блокировать очередь событий от ФС).

На одновременные I/O операции точно также накладывается ограничение в MAX_IO_THREADS=16 одновременных потоков, чтобы не создавать треды ОС неконтролируемым образом.

На каждую группу приемников создается «связка» горутин, которые занимаются отправкой данных.

Весь поток событий от файловой системы разбивается на «батчи» размером не более 1 Мб каждый, при этом чтение из файловой системы лимитируется 64 Кб на каждое прочитанное сообщение.

При необходимости доставки с помощью шардинга или с помощью round-robin на несколько серверов, используется подход, аналогичный «окну» TCP/IP — для всех отправленных, но пока что не заакноледженных, кусков, запоминается их offset и порядковый номер. Когда все «куски» для соответствующего сообщения доставлены, финальный offset записывается в файл.

Для шардинга каждое «сообщение» разбивается на несколько, и у соответствующего «пакета» запоминается число кусков, которые должны быть доставлены, перед тем, как можно будет записать offset.

Сообщения объединяются в «батчи» размером не более 1 Мб и посылаются с помощью GPB на центральный сервер. Если в течение 30 секунд ответа от центрального сервера не получено, то сервер на 60 секунд помечается, как оффлайн, и все события, которые предназначались для этого сервера, возвращаются «обратно» в общий канал для «передиспатчинга». Через 60 секунд делается ещё одна попытка отправить события на сервер, и так по кругу.

## Возможные проблемы и диагностика ошибок
Мониторинг для LSD достаточно прост и полностью аналогичен мониторингу для gosf логов — необходимо анализировать offsets.db и смотреть, какие смещения были успешно доставлены для каких файлов.

Если LSD «игнорирует» новые файлы и не удаляет старые, или удаляет старые «через раз», то это скорее всего говорит о том, что один из приемников событий «тупит» или вообще не работает. Из-за довольно сложной логики доставки событий на несколько хостов и сложного failover'а, возможны задержки доставки хостов на те машины, которые «тупят». В нормальных условиях такого происходить не должно — если такое происходит, то, возможно, на самом деле все хосты-приемники не справляются с нагрузкой.

Если вдруг возникают проблемы с занимаемым местом, и нужно удалять файлы, то нужно сначала остановить демон, сделать то, что нужно, и потом перезапустить его. Иначе у него в базе offsets.db останутся смещения для несуществующих файлов и возможна потеря событий, если появятся новые файлы с такими же inode — они будут читаться не сначала.

В целом, при любых проблемах демон можно просто перезапустить — прочитанные смещения сохраняются в файл раз в секунду, поэтому дублирование событий будет минимальным.

Демон может в теории «сожрать» много памяти (до нескольких гигабайт), если ему совсем некуда доставлять события в течение долгого времени — в таком случае перезапуск точно также решит проблему. Это небольшая недоработка архитектуры, которая, возможно, уже устранена к моменту, когда вы будете читать этот текст.

Не рекомендуется создавать слишком много файлов (десятки и сотни тысяч) — в этом нет необходимости, поскольку демон умеет читать файлы, в которые параллельно осуществляется запись. Также, на каждый файл, который нужно впоследствии удалять, создается по одной горутине, что расходует память (по 2 кб на файл).

Демон должен работать либо из-под рута, либо из-под пользователя, из-под которого работают остальные скрипты на машине (т.е. wwwrun), иначе у него не будет хватать прав сделать stat() на файлы, которые держатся открытыми у других процессов. Это может приводить к тому, что файлы удаляются раньше, чем перестают использоваться, что будет приводить к потере событий. При уровне NOTICE демон не пишет ничего в лог, когда у него нет прав на stat() — эти ошибки можно увидеть только с DEBUG уровнем.

Демон не должен жрать CPU «просто так», но в целом потребление CPU на клиентских машинах будет близко к scribe. На роутерах и на приемниках скорее всего потребление CPU будет значительно больше, чем у scribe, из-за того, что демон доставляет мелкие сообщения вместо крупных батчей, как у скрайба. В данный момент нет никаких конфигурационных опций, которые бы регулировали размеры сообщений, но по факту они должны подстраиваться автоматически — как только сервер начинает «затуплять», ему начинают присылаться пачки большего размера.

Рекомендуется запускать lsd-клиенты с ограничением на максимальный объем занимаемой памяти в размере 1 Гб RSS, а также GOMAXPROCS=4. На роутерах верхний лимит памяти будет зависеть от количества консьюмеров и от количества клиентов, примерно по следующей формуле:
```количество_клиентов*1 Мб + количество_консьюмеров*10 Мб + количество_категорий*1 Мб```

Нормальным является потребление памяти до нескольких Гб, что примерно соответствует аналогичным характеристикам у скрайба.

При доставке через кросс-ДЦ необходимо применение таких же трюков, как для scribe из-за модели «запрос-ответ» вместо «стриминга» событий. То есть, один и тот же сервер должен указываться много раз, чтобы было возможно полностью использовать полосу пропускания. Возможно, к моменту, когда вы это будете читать, lsd уже научился делать streaming, и этот костыль больше не нужен.
