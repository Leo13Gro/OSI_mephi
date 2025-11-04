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
2  varset MAX_CONN_RETRIES
2  varset MAX_DATA_RETRIES
0  varset conn_retries
0  varset data_retries
0  varset awaiting_data_ack
```


## T_CONNECT.REQ
```
;параметры:  address (число)
out("[INFO] T_CONNECT.REQ -> N_DATAGRAM.REQ")

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
out("N_datagram.ind")
unbufferit userdata user_data_type 1
out($user_data_type)

$user_data_type == $CONNECT_REQ 		if on_connect_req
$user_data_type == $CONNECT_CONF 		if on_connect_conf
$user_data_type == $DISCONNECT_REQ 		if on_disconnect_req
$user_data_type == $DATA_FRAME     		if on_data_frame
$user_data_type == $DATA_ACK			if on_data_ack
return

goto send_data

on_connect_req:
	out("N_DATAGRAM.IND [INFO]: CONNECT_REQ -> T_CONNECT.IND")
	T_CONNECT.IND generateup address $address
	return

on_connect_conf:
	out("N_DATAGRAM.IND [INFO]: CONNECT_CONF -> T_CONNECT.CONF")
	; получили подтверждение соединения — снимаем таймер и ретраи
	untimer $conn_timer_id
	0 varset conn_retries

	T_CONNECT.CONF generateup address $address
	return

on_disconnect_req:
	out("N_DATAGRAM.IND [INFO]: DISCONNECT_REQ -> T_DISCONNECT.IND")
	T_DISCONNECT.IND generateup address $address

	; очистим исходящую очередь и состояние передачи TODO:ресетнуть состояние передачи
	clearqueue out_q
	0 varset awaiting_data_ack
	return

on_data_frame:
	; Кадр: [type][seq] + payload
	sizeof(userdata) varset payload_len
	$payload_len - 2 varset payload_len

	unbufferit userdata user_data_type 1 in_seq 1 payload $payload_len

	; Если новые данные - прокидываем вверх
	$in_seq == $rcv_expect if fresh_frame
	
	; Если дубликат (ACK, отправленный ранее, потерялся) — НЕ отправляем вверх, но повторяем ACK
	out("N_DATAGRAM.IND [WARN]: caught duplicate DATA_FRAME, resend ACK")
	out($in_seq)
	out($rcv_expect)
	accept_data bufferit 2 $DATA_ACK 1 $in_seq 1
	N_DATAGRAM.REQ eventdown address $address userdata $accept_data
	return

fresh_frame:
	; Новый кадр — отдаём вверх и шлём ACK
	out("N_DATAGRAM.IND [INFO]: new DATA_FRAME deliver up, send ACK")
	T_DATA.IND generateup userdata $payload

	accept_data bufferit 2 $DATA_ACK 1 $in_seq 1
	N_DATAGRAM.REQ eventdown address $address userdata $accept_data

	($rcv_expect + 1) varset rcv_expect
	return

on_data_ack:
	; ACK: [type][seq]
	unbufferit userdata user_data_type 1 in_seq 1
	out("N_DATAGRAM.IND [INFO]: DATA_ACK seq=")
	out($in_seq)

	out("awaiting_data_ack")
	out($awaiting_data_ack)
	$awaiting_data_ack if maybe_accept_ack
	; Если нечего ждать — игнор (TODO: как будто всегда ждем?)
	return

maybe_accept_ack:
	; Снимаем таймер и продвигаем окно только если номер совпал с текущим исходящим seq
	out($snd_seq)
	$in_seq == $snd_seq if good_ack
	out("N_DATAGRAM.IND [WARN] unexpected ACK: got ")
	out($in_seq)
	out("expect ")
	out($snd_seq)
	return

good_ack:
	out("try untimer")
	untimer $data_timer_id
	0 varset awaiting_data_ack
	0 varset data_retries
	($snd_seq + 1) % 256 varset snd_seq

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
	dequeue (out_q) varset next_frame
	$next_frame varset last_data_frame
	N_DATAGRAM.REQ eventdown address $curr_address userdata $next_frame
	0 varset data_retries
	1 varset awaiting_data_ack
	DATA_TIMEOUT timer data_timer_id $RTO_DATA address $curr_address
	return
subend
```

## T_DISCONNECT.REQ
```
;параметры:  address (число)
out("T_DISCONNECT.REQ [INFO]: sending N_DATAGRAM.REQ")
disconnect_data bufferit 1 $DISCONNECT_REQ 1
N_DATAGRAM.REQ eventdown address $address userdata $disconnect_data
; TODO: добавить таймер
```

## T_CONNECT.RESP
```
;параметры:  address (число)
out("T_CONNECT.RESP [INFO]: sending N_DATAGRAM.REQ")
; Отправляем одно число - ENUM подтверждения
accept_data bufferit 1 $CONNECT_CONF 1

N_DATAGRAM.REQ eventdown address $address userdata $accept_data
```

## T_DATA.REQ
```
;параметры:   userdata (буфер)
out("T_DATA.REQ [INFO]: sending N_DATAGRAM.REQ")
out(sizeof(userdata))

; Один пакет - Кадр: [type=DATA_FRAME][seq][payload]
; Размер: 1 на тип, 1 на порядковый номер, n на данные === 2 + sizeof(...)
last_data_frame bufferit (2 + sizeof(userdata)) $DATA_FRAME 1 $snd_seq 1 $userdata sizeof(userdata)

; Складываем в исходящую очередь и пробуем отправить
queue out_q $last_data_frame
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
	dequeue (out_q) varset next_frame
	$next_frame varset last_data_frame
	N_DATAGRAM.REQ eventdown address $curr_address userdata $next_frame
	0 varset data_retries
	1 varset awaiting_data_ack
	DATA_TIMEOUT timer data_timer_id $RTO_DATA address $curr_address
	return
subend
```

# Таймеры

## CONNECT_TIMEOUT
```
; параметры: address (число)
$conn_retries + 1 varset conn_retries
$conn_retries <= $MAX_CONN_RETRIES if do_reconnect

; лимит превышен — индицируем разрыв как ошибку соединения
out("[ERROR] connect timeout exceeded")
T_DISCONNECT.IND generateup address $address
0 varset conn_retries
return

do_reconnect:
	out("[WARN] reconnect try:")
	out($conn_retries)
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

	out("DATA_TIMEOUT timer [ERROR]: DATA_TIMEOUT exceeded, give up and notify upper")
	T_DISCONNECT.IND generateup address $address
	0 varset data_retries
	0 varset awaiting_data_ack
	clearqueue out_q
	return

resend_data:
	out("DATA_TIMEOUT timer [WARN]: DATA retry:")
	out($data_retries)
	N_DATAGRAM.REQ eventdown address $address userdata $last_data_frame
	DATA_TIMEOUT timer data_timer_id $RTO_DATA address $address
	return
```