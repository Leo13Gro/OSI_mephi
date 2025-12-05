# Сеансовый уровень
## S_INIT
```
; ===== ENUM: типы сообщений сеансового уровня =====
MSG_CONNECT_REQ    declare integer
0 varset MSG_CONNECT_REQ

MSG_CONNECT_RESP   declare integer
1 varset MSG_CONNECT_RESP

MSG_U_ABORT        declare integer
2 varset MSG_U_ABORT

MSG_DATA           declare integer
3 varset MSG_DATA

MSG_SYNC_MAJOR_REQ declare integer
4 varset MSG_SYNC_MAJOR_REQ

MSG_SYNC_MAJOR_RESP declare integer
5 varset MSG_SYNC_MAJOR_RESP

MSG_GIVE_TOKENS   declare integer
6 varset MSG_GIVE_TOKENS

MSG_PLEASE_TOKENS declare integer
7 varset MSG_PLEASE_TOKENS

MSG_EXPEDITED_DATA declare integer
8 varset MSG_EXPEDITED_DATA

MSG_RELEASE       declare integer
9 varset MSG_RELEASE

; ===== Поля quality =====
def  declare integer
sync declare integer

; ===== Поле demand =====
marker declare integer

; ===== Для GIVE/PLEASE_TOKENS =====
req_token  declare integer

; ===== Глобальные переменные по соединению =====
curr_address        declare integer     ; адрес удалённой стороны

req_quality         declare buffer      ; quality, полученный или отправленный
req_demand          declare buffer      ; markers/demand

connection_params   declare buffer      ; буфер, отправляемый по транспорту

; ===== Флаги наличия маркеров =====
has_data_marker     declare integer
has_sync_marker     declare integer

; --- состояние упорядоченного разрыва ---
release_initiated   declare integer    ; мы отправили S_RELEASE.REQ
ordered_release     declare integer    ; мы участвуем в orderly-release как отвечающая сторона
0 varset release_initiated
0 varset ordered_release

msg_type declare integer

; ===== Передача данных =====
data_pdu 		declare buffer
payload 		declare buffer
payload_len 	declare integer
0 varset payload_len

; --- очередь исходящих данных, ждущих главной синхронизации ---
data_queue 		declare queue      ; очередь буферов userdata
tmp_userdata 	declare buffer     ; временный буфер для dequeue

; Коды ошибок
no_data_marker_error 	declare integer
1 varset no_data_marker_error

no_sync_marker_error 	declare integer
2 varset no_sync_marker_error

sync_error 				declare integer
3 varset sync_error
```

## S_CONNECT.REQ
```
;параметры:  address (число), quality (буфер), demand (буфер)
out CurrentSystemName() ": S_CONNECT.REQ [INFO]: sending T_CONNECT.REQ"

unbufferit quality 	def 1 	sync 1
unbufferit demand 	marker 1

;out "address = " $address
;out "def = " $def
;out "sync = " $sync
;out "marker = " $marker

$address 	varset curr_address
$quality 	varset req_quality
$demand 	varset req_demand

; Формируем S-PDU CONNECT_REQ:
; [MSG_CONNECT_REQ][def][sync][marker]
connection_params bufferit 4 $MSG_CONNECT_REQ 1 $def 1 $sync 1 $marker 1

; Выставляем флаги маркеров
($marker == 1) || ($marker == 3) varset has_data_marker
($marker == 1) || ($marker == 4) varset has_sync_marker

T_CONNECT.REQ eventdown address $address
return
```

## T_CONNECT.IND
```
;параметры:  address (число)
out CurrentSystemName() ": T_CONNECT.IND [INFO]: incoming transport connect from " $address
$address varset curr_address

out CurrentSystemName() ": T_CONNECT.IND [INFO]: sending T_CONNECT.RESP"
T_CONNECT.RESP eventdown address $address

```

## T_CONNECT.CONF
```
;параметры: address (число)
out CurrentSystemName() ": T_CONNECT.CONF [INFO]: transport connected, sending connection params"

T_DATA.REQ eventdown userdata $connection_params
```

## S_CONNECT.RESP
```
;параметры: address (число), quality (буфер)
out CurrentSystemName() ": S_CONNECT.RESP [INFO]: connection accepted, sending CONNECT.RESP (with params)"

unbufferit quality def 1 sync 1

; Формируем ответный S-PDU:
; [MSG_CONNECT_RESP][def][sync]
connection_params bufferit 3 $MSG_CONNECT_RESP 1 $def 1 $sync 1

T_DATA.REQ eventdown userdata $connection_params

return
```

## T_DATA.IND
```
;параметры:  userdata (буфер)
out CurrentSystemName() ": T_DATA.IND [INFO]: received userdata"

; Распаковываем тип сообщения
unbufferit userdata msg_type 1

out CurrentSystemName() ": T_DATA.IND [INFO]: msg_type = " $msg_type

$msg_type == $MSG_CONNECT_REQ 		if handle_connect_req
$msg_type == $MSG_CONNECT_RESP 		if handle_connect_resp
$msg_type == $MSG_U_ABORT 			if handle_u_abort
$msg_type == $MSG_DATA 				if handle_data
$msg_type == $MSG_SYNC_MAJOR_REQ 	if handle_sync_major_req
$msg_type == $MSG_SYNC_MAJOR_RESP 	if handle_sync_major_resp
$msg_type == $MSG_GIVE_TOKENS 		if handle_give_tokens
$msg_type == $MSG_PLEASE_TOKENS 	if handle_please_tokens
$msg_type == $MSG_EXPEDITED_DATA 	if handle_expedited_data
$msg_type == $MSG_RELEASE 			if handle_release

out CurrentSystemName() ": T_DATA.IND [ERROR]: unknown session message type = " $msg_type
return

; ------------------------------------------
handle_connect_req:
	; PDU формата: [msg_type][def][sync][marker]
	unbufferit userdata msg_type 1 def 1 sync 1 marker 1

	out CurrentSystemName() ": T_DATA.IND [INFO]: CONNECT_REQ(def=" $def ", sync=" $sync ", marker=" $marker ")"

	; собираем quality/demand
	req_quality bufferit 2 $def 1 $sync 1
	req_demand  bufferit 1 $marker 1

	; Выставляем флаги маркеров
	($marker == 2) || ($marker == 4) varset has_data_marker
	($marker == 2) || ($marker == 3) varset has_sync_marker

	; теперь можно выдать сеансовую индикацию
	S_CONNECT.IND generateup address $curr_address quality $req_quality demand $req_demand

	return
; ------------------------------------------
handle_connect_resp:
	; PDU формата: [msg_type][def][sync]
	unbufferit userdata msg_type 1 def 1 sync 1

	out CurrentSystemName() ": T_DATA.IND [INFO]: CONNECT_RESP(def=" $def ", sync=" $sync ")"

	req_quality bufferit 2 $def 1 $sync 1

	; Проверка: это инициатор, завершаем соединение
	S_CONNECT.CONF generateup address $curr_address quality $req_quality

	return
; ---------- U_ABORT: завершение по инициативе удалённого пользователя ----------
handle_u_abort:
    out CurrentSystemName() ": T_DATA.IND [INFO]: got U_ABORT from peer – notify user and disconnect"

    ; Уведомляем локального пользователя: удалённая сторона оборвала сеанс
    S_U_ABORT.IND generateup

    ; Инициируем разрыв транспортного соединения
    ;T_DISCONNECT.REQ eventdown address $curr_address

    return

; ------------------------------------------
handle_data:
    out CurrentSystemName() ": T_DATA.IND [INFO]: DATA message received"

    ; PDU формата:
    ; [msg_type][userdata]
    unbufferit userdata msg_type 1 payload (sizeof(userdata) - 1)

	out CurrentSystemName() ": T_DATA.IND [INFO]: enqueue DATA payload until SYNC_MAJOR completed"
    queue data_queue $payload

    ; Передаём полезные данные наверх
    ; S_DATA.IND generateup userdata $payload

    return

; ---------- SYNC_MAJOR.REQ: пришёл запрос главной синхронизации ----------
handle_sync_major_req:
    ; PDU: [MSG_SYNC_MAJOR_REQ][sync]
    unbufferit userdata msg_type 1 sync 1

    out CurrentSystemName() ": T_DATA.IND [INFO]: SYNC_MAJOR_REQ(sync=" $sync ")"

    ; при необходимости можно сохранить sync в локальное состояние
    ; $sync varset sync   ; если хочешь обновлять глобальную переменную

    ; уведомляем уровень представления
    S_SYNC_MAJOR.IND generateup

    return

; ---------- SYNC_MAJOR.RESP: ответ на наш запрос ----------
handle_sync_major_resp:
    ; PDU: [MSG_SYNC_MAJOR_RESP][sync]
    unbufferit userdata msg_type 1 sync 1

    out CurrentSystemName() ": T_DATA.IND [INFO]: SYNC_MAJOR_RESP(sync=" $sync ")"

    ; при необходимости обновляем локальное значение sync
    ; $sync varset sync

    ; уведомляем представление, что синхронизация завершена
    S_SYNC_MAJOR.CONF generateup

    return

; GIVE_TOKENS: удалённая сторона отдаёт маркер
handle_give_tokens:
    ; PDU: [MSG_GIVE_TOKENS][req_token]
    unbufferit userdata msg_type 1 req_token 1

    out CurrentSystemName() ": T_DATA.IND [INFO]: GIVE_TOKENS(req_token=" $req_token ")"

    ; token = 1 — маркер данных
    ; token = 2 — маркер синхронизации

    $req_token == 1 if give_data_token_recv
    $req_token == 2 if give_sync_token_recv
    return

give_data_token_recv:
    ; если маркер данных уже есть — ошибка дубликата
    $has_data_marker != 0 if give_data_token_dup

    1 varset has_data_marker
    S_GIVE_TOKENS.IND generateup token $req_token
    return

give_data_token_dup:
    out CurrentSystemName() ": T_DATA.IND [ERROR]: duplicate DATA token"
    S_P_EXCEPTION.IND generateup error $sync_error
    return

give_sync_token_recv:
    ; если маркер синхронизации уже есть — ошибка дубликата
    $has_sync_marker != 0 if give_sync_token_dup

    1 varset has_sync_marker
    S_GIVE_TOKENS.IND generateup token $req_token
    return

give_sync_token_dup:
    out CurrentSystemName() ": T_DATA.IND [ERROR]: duplicate SYNC token"
    S_P_EXCEPTION.IND generateup error $sync_error
	return

; ---------- PLEASE_TOKENS: удалённая сторона просит маркер ----------
handle_please_tokens:
    ; PDU: [MSG_PLEASE_TOKENS][req_token]
    unbufferit userdata msg_type 1 req_token 1

    out CurrentSystemName() ": T_DATA.IND [INFO]: PLEASE_TOKENS(req_token=" $req_token ")"

    ; Просто передаём запрос наверх.
    ; Решение, отдавать ли маркер, принимает пользователь (через S_GIVE_TOKENS.REQ).
    S_PLEASE_TOKENS.IND generateup token $req_token

    return

; ---------- EXPEDITED_DATA: срочные данные ----------
handle_expedited_data:
    out CurrentSystemName() ": T_DATA.IND [INFO]: EXPEDITED_DATA message received"

    ; PDU формата: [MSG_EXPEDITED_DATA][userdata]
    unbufferit userdata msg_type 1 payload (sizeof(userdata) - 1)

    ; Срочные данные — сразу наверх, без очередей и ожидания синхронизации
    S_EXPEDITED_DATA.IND generateup userdata $payload

    return

; ---------- RELEASE: упорядоченный разрыв соединения ----------
handle_release:
    ; PDU формата: [MSG_RELEASE]
    ; msg_type уже извлечён, можно просто залогировать
    out CurrentSystemName() ": T_DATA.IND [INFO]: RELEASE PDU received – notify user"

    ; Индикация упорядоченного разрыва наверх
    S_RELEASE.IND generateup

    return
```

## S_U_ABORT.REQ
```
;параметры:  нет
out CurrentSystemName() ": S_U_ABORT.REQ [INFO]: user requested abort"

; Формируем сеансовый PDU:
; [MSG_U_ABORT]
connection_params bufferit 1 $MSG_U_ABORT 1

out CurrentSystemName() ": S_U_ABORT.REQ [INFO]: sending U_ABORT PDU via T_DATA.REQ"
T_DATA.REQ eventdown userdata $connection_params

; После отправки уведомления – сразу рвём транспортное соединение
out CurrentSystemName() ": S_U_ABORT.REQ [INFO]: call T_DISCONNECT.REQ"
T_DISCONNECT.REQ eventdown address $curr_address

return
```

## T_DISCONNECT.IND
```
;параметры: нет
out CurrentSystemName() ": T_DISCONNECT.IND [INFO]: transport disconnected"

; Проверяем, упорядоченный разрыв или нет

$release_initiated 	!= 0 if release_orderly_initiator
$ordered_release 	!= 0 if release_orderly_responder

; Иначе – это аварийный разрыв поставщиком (S_P_ABORT.IND)
out CurrentSystemName() ": T_DISCONNECT.IND [WARN]: unexpected disconnect – provider abort"
S_P_ABORT.IND generateup

; сбрасываем флаги на всякий случай
0 varset release_initiated
0 varset ordered_release

return

; --- инициирующая сторона: выдаём S_RELEASE.CONF ---
release_orderly_initiator:
    out CurrentSystemName() ": T_DISCONNECT.IND [INFO]: orderly release completed, send S_RELEASE.CONF"
    S_RELEASE.CONF generateup

    0 varset release_initiated
    0 varset ordered_release
    return

; --- отвечающая сторона: уже видела S_RELEASE.IND/RESP ---
release_orderly_responder:
    out CurrentSystemName() ": T_DISCONNECT.IND [INFO]: disconnect after orderly release (responder)"

    0 varset ordered_release
    0 varset release_initiated
    return

```

## S_DATA.REQ
```
;параметры:   userdata (буфер)
break

; ----- Проверка наличия маркера данных -----
$has_data_marker == 0 if no_data_token

out CurrentSystemName() ": S_DATA.REQ [INFO]: sending user data"

; Формируем PDU для данных:
; [MSG_DATA][userdata]
data_pdu bufferit sizeof(userdata) + 1 $MSG_DATA 1 $userdata sizeof(userdata)

T_DATA.REQ eventdown userdata $data_pdu
return

no_data_token:
	out CurrentSystemName() ": S_DATA.REQ [ERROR]: no data marker, cannot send data"
	S_P_EXCEPTION.IND generateup error $no_data_marker_error
    return
```

## S_SYNC_MAJOR.REQ
```
;параметры:  нет
break
out CurrentSystemName() ": S_SYNC_MAJOR.REQ [INFO]: request major sync, sync = " $sync

; ----- Проверка наличия маркера синхронизации -----
$has_sync_marker == 0 if no_sync_token

; Формируем PDU:
; [MSG_SYNC_MAJOR_REQ][sync]
connection_params bufferit 2 $MSG_SYNC_MAJOR_REQ 1 $sync 1

out CurrentSystemName() ": S_SYNC_MAJOR.REQ [INFO]: send SYNC_MAJOR_REQ via T_DATA.REQ"
T_DATA.REQ eventdown userdata $connection_params
return

no_sync_token:
	out "S_SYNC_MAJOR.REQ [ERROR]: no sync marker, cannot set major sync"
    S_P_EXCEPTION.IND generateup error $no_sync_marker_error
    return
```

## S_SYNC_MAJOR.RESP
```
;параметры:  нет
out CurrentSystemName() ": S_SYNC_MAJOR.RESP [INFO]: user accepted major sync, sync = " $sync

; Формируем PDU:
; [MSG_SYNC_MAJOR_RESP][sync]
connection_params bufferit 2 $MSG_SYNC_MAJOR_RESP 1 $sync 1

; ----- выдаём все накопленные DATA вверх -----
flush_queue:
    qcount (data_queue) == 0 if flush_done

    ; забираем первый накопленный payload
    dequeue (data_queue) varset payload

    out CurrentSystemName() ": S_SYNC_MAJOR.RESP [INFO]: delivering queued DATA to user"
    S_DATA.IND generateup userdata $payload
	
	goto flush_queue

flush_done:
	T_DATA.REQ eventdown userdata $connection_params
	return

return
```

## S_GIVE_TOKENS.REQ
```
;параметры:  token (число)
; token: 1 - маркер данных, 2 - маркер синхронизации
out CurrentSystemName() ": S_GIVE_TOKENS.REQ [INFO]: giving token " $token

; token = 1 — отдаём маркер данных
; token = 2 — отдаём маркер синхронизации
$token == 1 if give_data_token_send
$token == 2 if give_sync_token_send
goto give_tokens_req_build

give_data_token_send:
    ; нет маркера данных — ошибка
    $has_data_marker == 0 if give_token_no_data
    0 varset has_data_marker
    goto give_tokens_req_build

give_sync_token_send:
    ; нет маркера синхронизации — ошибка
    $has_sync_marker == 0 if give_token_no_sync
    0 varset has_sync_marker
    goto give_tokens_req_build

give_token_no_data:
    out CurrentSystemName() ": S_GIVE_TOKENS.REQ [ERROR]: no DATA token to give"
    S_P_EXCEPTION.IND generateup error $sync_error
    return

give_token_no_sync:
    out CurrentSystemName() ": S_GIVE_TOKENS.REQ [ERROR]: no SYNC token to give"
    S_P_EXCEPTION.IND generateup error $sync_error
    return

give_tokens_req_build:
    ; Формируем PDU: [MSG_GIVE_TOKENS][token]
    connection_params bufferit 2 $MSG_GIVE_TOKENS 1 $token 1

    out CurrentSystemName() ": S_GIVE_TOKENS.REQ [INFO]: sending GIVE_TOKENS via T_DATA.REQ"
    T_DATA.REQ eventdown userdata $connection_params

    return
```

## S_PLEASE_TOKENS.REQ
```
;параметры:  token (число)
; token: 1 - маркер данных, 2 - маркер синхронизации
out CurrentSystemName() ": S_PLEASE_TOKENS.REQ [INFO]: requesting token " $token

; Формируем PDU: [MSG_PLEASE_TOKENS][token]
connection_params bufferit 2 $MSG_PLEASE_TOKENS 1 $token 1

out CurrentSystemName() ": S_PLEASE_TOKENS.REQ [INFO]: sending PLEASE_TOKENS via T_DATA.REQ"
T_DATA.REQ eventdown userdata $connection_params

return
```

## S_EXPEDITED_DATA.REQ
```
;параметры:   userdata (буфер)
out CurrentSystemName() ": S_EXPEDITED_DATA.REQ [INFO]: sending expedited user data"

; Срочные данные не зависят от маркеров и синхронизации — отправляем сразу.

; Формируем PDU для срочных данных:
; [MSG_EXPEDITED_DATA][userdata]
data_pdu bufferit sizeof(userdata) + 1 $MSG_EXPEDITED_DATA 1 $userdata sizeof(userdata)

T_DATA.REQ eventdown userdata $data_pdu

return
```

## S_RELEASE.REQ
```
;параметры:  нет
out CurrentSystemName() ": S_RELEASE.REQ [INFO]: orderly release requested by user"

; помечаем, что инициатива за нами
1 varset release_initiated

; Формируем PDU: [MSG_RELEASE]
connection_params bufferit 1 $MSG_RELEASE 1

out CurrentSystemName() ": S_RELEASE.REQ [INFO]: sending RELEASE PDU via T_DATA.REQ"
T_DATA.REQ eventdown userdata $connection_params

return
```

## S_RELEASE.RESP
```
;параметры:  нет
out CurrentSystemName() ": S_RELEASE.RESP [INFO]: user accepted orderly release"

; ----- выдаём все накопленные DATA вверх -----
flush_queue:
    qcount (data_queue) == 0 if flush_done

    ; забираем первый накопленный payload
    dequeue (data_queue) varset payload

    out CurrentSystemName() ": S_RELEASE.RESP [INFO]: delivering queued DATA to user"
    S_DATA.IND generateup userdata $payload
	
	goto flush_queue

flush_done:
	; помечаем, что разрыв соединения – упорядоченный (со стороны отвечающей системы)
	1 varset ordered_release

	; инициируем разрыв транспортного соединения
	out CurrentSystemName() ": S_RELEASE.RESP [INFO]: calling T_DISCONNECT.REQ"
	T_DISCONNECT.REQ eventdown address $curr_address
	return

return
```

## 
```

```
# Транспортный уровень
## T_INIT
```
connect_data declare buffer
disconnect_data declare buffer
accept_data declare buffer
user_data_type declare integer
curr_address declare integer

; Для DATA/ACK
last_data_frame		declare buffer
awaiting_data_ack	declare integer
snd_seq				declare integer    ; номер следующего исходящего кадра (0..255)
rcv_expect			declare integer    ; ожидаемый номер кадра на приёме (0..255)
in_seq				declare integer
payload_len			declare integer
payload				declare buffer
next_frame			declare buffer
out_q				declare queue

; Формат кадров: 
; DATA_FRAME: [type=3][seq=1 байт][len=2 байта][payload len байт]
; DATA_ACK: [type=4][seq=1 байт]
; первый байт в userdata — тип (Enum).

; ENUM - переменные для определения типа реквеста
CONNECT_REQ declare integer
CONNECT_CONF declare integer
DISCONNECT_REQ declare integer
DATA_FRAME declare integer
DATA_ACK declare integer

; Инициализация
0 varset CONNECT_REQ
1 varset CONNECT_CONF
2 varset DISCONNECT_REQ
3 varset DATA_FRAME
4 varset DATA_ACK

; Переменные под таймеры/ретраи:
conn_timer_id declare integer
data_timer_id declare integer
conn_retries declare integer
data_retries declare integer
RTO_CONNECT declare integer
RTO_DATA declare integer
MAX_CONN_RETRIES declare integer
MAX_DATA_RETRIES declare integer

; Инициализация
80 varset RTO_CONNECT
80 varset RTO_DATA
6  varset MAX_CONN_RETRIES
6  varset MAX_DATA_RETRIES
0  varset conn_retries
0  varset data_retries
0  varset awaiting_data_ack
```


## T_CONNECT.REQ
```
;параметры:  address (число)

0 varset snd_seq
0 varset rcv_expect
0 varset awaiting_data_ack
0 varset data_retries
clearqueue out_q
untimer $data_timer_id

out CurrentSystemName() ": [INFO] T_CONNECT.REQ -> N_DATAGRAM.REQ"

; Сохраняем текущий адрес
$address varset curr_address

; Отправляем одно число - ENUM Connect
connect_data bufferit 1 $CONNECT_REQ 1
N_DATAGRAM.REQ eventdown address $address userdata $connect_data

; Запускаем таймер повтора CONNECT (повторяем предыдущую отправку)
CONNECT_TIMEOUT timer conn_timer_id $RTO_CONNECT address $address
```

## N_DATAGRAM.IND
```
;параметры:  address (число), userdata (буфер)
unbufferit userdata user_data_type 1
out CurrentSystemName() ": N_DATAGRAM.IND [INFO]: got data with type = " $user_data_type
$address varset curr_address

$user_data_type == $CONNECT_REQ 		if on_connect_req
$user_data_type == $CONNECT_CONF 		if on_connect_conf
$user_data_type == $DISCONNECT_REQ 		if on_disconnect_req
$user_data_type == $DATA_FRAME     		if on_data_frame
$user_data_type == $DATA_ACK			if on_data_ack
return

goto send_data

on_connect_req:
	; обнуляем переменные
	0 varset snd_seq
	0 varset rcv_expect
	0 varset awaiting_data_ack
	0 varset data_retries
	clearqueue out_q
	untimer $data_timer_id
	
	out CurrentSystemName() ": N_DATAGRAM.IND [INFO]: CONNECT_REQ -> T_CONNECT.IND"
	T_CONNECT.IND generateup address $address
	return

on_connect_conf:
	out CurrentSystemName() ": N_DATAGRAM.IND [INFO]: CONNECT_CONF -> T_CONNECT.CONF"
	; получили подтверждение соединения — снимаем таймер и ретраи
	untimer $conn_timer_id
	0 varset conn_retries

	T_CONNECT.CONF generateup address $address
	return

on_disconnect_req:
	out CurrentSystemName() ": N_DATAGRAM.IND [INFO]: DISCONNECT_REQ -> T_DISCONNECT.IND"
	T_DISCONNECT.IND generateup address $address

	; очистим исходящую очередь и состояние передачи
	clearqueue out_q
	0 varset awaiting_data_ack
	0 varset snd_seq
	0 varset rcv_expect
	0 varset data_retries
	0 varset conn_retries
	untimer $data_timer_id
	untimer $conn_timer_id

	return

on_data_frame:
	; Кадр: [type][seq] + payload
	sizeof(userdata) varset payload_len
	$payload_len - 2 varset payload_len

	unbufferit userdata user_data_type 1 in_seq 1 payload $payload_len

	; Если новые данные - прокидываем вверх
	$in_seq == $rcv_expect if fresh_frame
	
	; Если дубликат (ACK, отправленный ранее, потерялся) — НЕ отправляем вверх, но повторяем ACK
	out CurrentSystemName() "N_DATAGRAM.IND [WARN]: caught duplicate DATA_FRAME, resend ACK"
	accept_data bufferit 2 $DATA_ACK 1 $in_seq 1
	N_DATAGRAM.REQ eventdown address $address userdata $accept_data
	return

fresh_frame:
	; Новый кадр — отдаём вверх и шлём ACK
	out CurrentSystemName() ": N_DATAGRAM.IND [INFO]: new DATA_FRAME deliver up, send ACK"
	T_DATA.IND generateup userdata $payload

	accept_data bufferit 2 $DATA_ACK 1 $in_seq 1
	N_DATAGRAM.REQ eventdown address $address userdata $accept_data

	($rcv_expect + 1) varset rcv_expect
	return

on_data_ack:
	; ACK: [type][seq]
	unbufferit userdata user_data_type 1 in_seq 1
	out CurrentSystemName() ": N_DATAGRAM.IND [INFO]: DATA_ACK seq=" $in_seq
	out CurrentSystemName() ": awaiting_data_ack = " $awaiting_data_ack
	$awaiting_data_ack if maybe_accept_ack
	; Если нечего ждать — игнор (TODO: как будто всегда ждем?)
	return

maybe_accept_ack:
	; Снимаем таймер и продвигаем окно только если номер совпал с текущим исходящим seq
	$in_seq == $snd_seq if good_ack
	out CurrentSystemName() ": N_DATAGRAM.IND [WARN] unexpected ACK: got " $in_seq " expect " $snd_seq

	return

good_ack:
	out CurrentSystemName() ": N_DATAGRAM.IND [INFO]: untimer"
	untimer $data_timer_id
	0 varset awaiting_data_ack
	0 varset data_retries
	$snd_seq + 1 varset snd_seq

	; отправим следующий из очереди, если есть
	subprog maybe_send_next
	return

; Подпрограмма - отправка следующего кадра из очереди
return
substart maybe_send_next
$awaiting_data_ack if msn_busy
qcount (out_q) varset payload_len
$payload_len > 0 if msn_send
return

msn_busy:
	return

msn_send:
	; достаём следующий payload для отправки
	dequeue (out_q) varset payload

	; формируем кадр с АКТУАЛЬНЫМ snd_seq
	; Один пакет - Кадр: [type=DATA_FRAME][seq][payload]
	; Размер: 1 на тип, 1 на порядковый номер, n на данные === 2 + sizeof(...)
	last_data_frame bufferit (2 + sizeof(payload)) $DATA_FRAME 1 $snd_seq 1 $payload sizeof(payload)

	out CurrentSystemName() ": T_DATA.REQ [INFO]: send to address = " $curr_address " seq = " $snd_seq
	N_DATAGRAM.REQ eventdown address $curr_address userdata $last_data_frame
	0 varset data_retries
	1 varset awaiting_data_ack
	DATA_TIMEOUT timer data_timer_id $RTO_DATA address $curr_address
	return
subend
```

## T_DISCONNECT.REQ
```
;параметры:  address (число)
out CurrentSystemName() ":T_DISCONNECT.REQ [INFO]: sending N_DATAGRAM.REQ"
disconnect_data bufferit 1 $DISCONNECT_REQ 1
N_DATAGRAM.REQ eventdown address $address userdata $disconnect_data
; TODO: добавить таймер
```

## T_CONNECT.RESP
```
;параметры:  address (число)
out CurrentSystemName() ": T_CONNECT.RESP [INFO]: sending N_DATAGRAM.REQ"
; Отправляем одно число - ENUM подтверждения
accept_data bufferit 1 $CONNECT_CONF 1

N_DATAGRAM.REQ eventdown address $address userdata $accept_data
```

## T_DATA.REQ
```
;параметры:   userdata (буфер)
out CurrentSystemName() ": T_DATA.REQ [INFO]: got data with size = " sizeof(userdata)
out CurrentSystemName() ": T_DATA.REQ [INFO]: sending N_DATAGRAM.REQ"

; Складываем в исходящую очередь исходные данные и пробуем отправить (упаковка в отправке)
queue out_q $userdata
subprog maybe_send_next
return

; В подпрограмме так отправляем: N_DATAGRAM.REQ eventdown address $curr_address userdata $userdata
; Подпрограмма - отправка следующего кадра из очереди
return
substart maybe_send_next
$awaiting_data_ack if msn_busy
qcount (out_q) varset payload_len
$payload_len > 0 if msn_send
return

msn_busy:
	return

msn_send:
	; достаём следующий payload для отправки
	dequeue (out_q) varset payload

	; формируем кадр с АКТУАЛЬНЫМ snd_seq
	; Один пакет - Кадр: [type=DATA_FRAME][seq][payload]
	; Размер: 1 на тип, 1 на порядковый номер, n на данные === 2 + sizeof(...)
	last_data_frame bufferit (2 + sizeof(payload)) $DATA_FRAME 1 $snd_seq 1 $payload sizeof(payload)

	out CurrentSystemName() ": T_DATA.REQ [INFO]: send to address = " $curr_address " seq = " $snd_seq
	N_DATAGRAM.REQ eventdown address $curr_address userdata $last_data_frame
	0 varset data_retries
	1 varset awaiting_data_ack
	DATA_TIMEOUT timer data_timer_id $RTO_DATA address $curr_address
	return
subend
```

## CONNECT_TIMEOUT
```
; параметры: address (число)
$conn_retries + 1 varset conn_retries
$conn_retries <= $MAX_CONN_RETRIES if do_reconnect

; лимит превышен — индицируем разрыв как ошибку соединения
out CurrentSystemName() ": [ERROR] connect timeout exceeded"
T_DISCONNECT.IND generateup address $address
0 varset conn_retries
0 varset snd_seq
0 varset rcv_expect
0 varset awaiting_data_ack
0 varset data_retries
clearqueue out_q
untimer $data_timer_id

return

do_reconnect:
	out CurrentSystemName() ": [WARN] reconnect try: " $conn_retries
	; переиспользуем готовый connect_data (глобальный буфер)
	N_DATAGRAM.REQ eventdown address $address userdata $connect_data
	CONNECT_TIMEOUT timer conn_timer_id $RTO_CONNECT address $address
	return
```

## DATA_TIMEOUT
```
; параметры: address (число)
$awaiting_data_ack if do_redo
; если уже не ждём — ничего
return

do_redo:
	$data_retries + 1 varset data_retries
	$data_retries <= $MAX_DATA_RETRIES if resend_data

	out CurrentSystemName() ": DATA_TIMEOUT timer [ERROR]: DATA_TIMEOUT exceeded, give up and notify upper"
	T_DISCONNECT.IND generateup address $address

	0 varset data_retries
	0 varset awaiting_data_ack
	0 varset snd_seq
	0 varset rcv_expect
	0 varset conn_retries
	clearqueue out_q
	untimer $data_timer_id

	return

resend_data:
	out CurrentSystemName() ": DATA_TIMEOUT timer [WARN]: DATA retry: " $data_retries
	N_DATAGRAM.REQ eventdown address $address userdata $last_data_frame
	DATA_TIMEOUT timer data_timer_id $RTO_DATA address $address
	return
```