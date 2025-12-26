# Прикладной уровень

## A_INIT.REQ
```
; внутренние переменные
receiver_name declare string
initiator_name declare string
saved_address declare integer
guide_buffer declare buffer
name_to_resolve   declare string

curr_system_name declare string
curr_context declare buffer
curr_quality declare buffer

data_type declare integer
pack_type declare integer
d_type declare integer

; GS переменные
gs_demand declare buffer
gs_demand bufferit 1 1 1

gs_context declare buffer
gs_context bufferit 4 "{" 1 "}" 1  1 1  1 1

; ---------- ассоциации ----------
asos_apcon   declare string
asos_demand  declare buffer

; ---------- локгайд поля ----------
guide_id   declare integer
guide_address   declare integer
guide_time_1  declare integer
guide_time_2  declare integer
guide_char_1  declare integer
guide_char_2  declare integer

; ---------- временные буферы ----------
buffer_send       declare buffer
receiver_buffer      declare buffer
initiator_buffer      declare buffer
buffer_tmp declare buffer ;TODO: change

; Очередь
data_queue declare queue

; --- timers (используются ниже) ---
t_retry declare integer
t_resolved declare integer
t_dt declare integer
t_gs_retry declare integer
t_term_req declare integer
t_term_resp declare integer
t_resolve declare integer
t_gs declare integer
t_gs_data declare integer
t_ declare integer

; ENUM id ответа справочника
GOT_ADDRESS declare integer
0 varset GOT_ADDRESS

ADDRESS_AFTER_TIME_1 declare integer
1 varset ADDRESS_AFTER_TIME_1

ADDRESS_BETWEEN_TIME_1_AND_2 declare integer
2 varset ADDRESS_BETWEEN_TIME_1_AND_2

NO_ADDRESS declare integer
3 varset NO_ADDRESS

UNKNOWN_NAME_MAY_BE_REPLACE declare integer
4 varset UNKNOWN_NAME_MAY_BE_REPLACE

; --- expedited enum ---
PACK_TYPE_EXP declare integer
3 varset PACK_TYPE_EXP

CON_REQ_TYPE declare integer
1 varset CON_REQ_TYPE

CON_RES_TYPE declare integer
2 varset CON_RES_TYPE
```

## A_TERMINATE.REQ
```
;параметры: нет
out CurrentSystemName() ": A_TERMINATE.REQ [INFO]: terminate -> release apcon = " $asos_apcon
A_RELEASE.REQ timer t_term_req 0 apcon $asos_apcon
return
```

## A_RELEASE.REQ
```
;параметры: apcon (строка)
out CurrentSystemName() ": A_RELEASE.REQ [INFO]: got string = " $apcon ", down P_RELEASE.REQ"
P_RELEASE.REQ eventdown
```

## P_RELEASE.CONF
```
;параметры:  нет
out CurrentSystemName() ": P_RELEASE.CONF [INFO]: apcon = " $asos_apcon
$asos_apcon == "DT" if handle_dt
$asos_apcon == "GS" if handle_gs
return

handle_dt:
	A_TERMINATE.CONF generateup
	return

handle_gs:
	out CurrentSystemName() ": P_RELEASE.CONF [INFO]: GS released -> continue DT with resolved address"
	A_RESOLVE.IND.DT timer t_ 0 buffer_with_address $guide_buffer
	return
```

## A_TRANSFER_INIT.REQ
```
;параметры: namer (строка), namei (строка), context (буфер), quality (буфер)
out CurrentSystemName() ": A_TRANSFER_INIT.REQ [INFO]: namer=" $namer " namei=" $namei

$namer   varset receiver_name
$namei   varset initiator_name
$context varset curr_context
$quality varset curr_quality

A_RESOLVE.REQ timer t_resolve 0 name $receiver_name
return
```

## A_RESOLVE.REQ
```
;параметры: name (строка)
out CurrentSystemName() ": A_RESOLVE.REQ [INFO]: name = " $name

$saved_address != 0 if resolve_saved

locguide($name) varset guide_buffer
unbufferit guide_buffer guide_id 1 guide_address 1 guide_time_1 1 guide_time_2 1 guide_char_1 1 guide_char_2 1

out CurrentSystemName() ": A_RESOLVE.REQ [DEBUG]: type=" $guide_id " address = " $guide_address " time1=" $guide_time_1 " time2 = " $guide_time_2 " char1=" $guide_char_1 " char2 = " $guide_char_2

$guide_id == $GOT_ADDRESS 					if handle_address
$guide_id == $ADDRESS_AFTER_TIME_1 			if wait_for_address
$guide_id == $ADDRESS_BETWEEN_TIME_1_AND_2 	if wait_for_address
$guide_id == $NO_ADDRESS 					if get_guide
$guide_id == $UNKNOWN_NAME_MAY_BE_REPLACE 	if get_guide
return

handle_address:
	; Сохраняем адрес
	$guide_address varset saved_address
	; Кладем адрес, 0, " ", " "
	; буффер: [guide_address][0][ ][ ]
	guide_buffer bufferit 4 $guide_address 1 0 1 " " 1 " " 1
	
	A_RESOLVE.IND.DT timer t_resolved 0 buffer_with_address $guide_buffer
	
	return

wait_for_address:
	A_RESOLVE.REQ timer t_retry $guide_time_1 name $name
	return

get_guide:
	out CurrentSystemName() ": A_RESOLVE.REQ [INFO]: timer A_ASSOCIATE.REQ"
	$name varset name_to_resolve

	locguide("Guide") varset guide_buffer
	unbufferit guide_buffer guide_id 1 guide_address 1 guide_time_1 1 guide_time_2 1 guide_char_1 1 guide_char_2 1
	out CurrentSystemName() ": A_RESOLVE.REQ [DEBUG]: type=" $guide_id " address = " $guide_address " time1=" $guide_time_1 " time2 = " $guide_time_2 " char1=" $guide_char_1 " char2 = " $guide_char_2
	receiver_buffer bufferit 6 $guide_address 1 "Guide" 5

	CurrentSystemName() varset curr_system_name
	locguide($curr_system_name) varset guide_buffer
	unbufferit guide_buffer guide_id 1 guide_address 1 guide_time_1 1 guide_time_2 1 guide_char_1 1 guide_char_2 1
	out CurrentSystemName() ": A_RESOLVE.REQ [DEBUG]: type=" $guide_id " address = " $guide_address " time1=" $guide_time_1 " time2 = " $guide_time_2 " char1=" $guide_char_1 " char2 = " $guide_char_2
	initiator_buffer bufferit 1+sizeof(curr_system_name) $guide_address 1 $curr_system_name sizeof(curr_system_name)

	A_ASSOCIATE.REQ timer t_gs 0 namer $receiver_buffer namei $initiator_buffer quality $curr_quality demand $gs_demand context $gs_context apcon "GS"
	return

resolve_saved:
	guide_buffer bufferit 4 $saved_address 1 0 1 " " 1 " " 1
	A_RESOLVE.IND.DT timer t_ 0 buffer_with_address $guide_buffer
	return
```

## A_ASSOCIATE.REQ
```
;параметры: namer (буфер), namei (буфер), quality (буфер), demand (буфер), context (буфер), apcon (строка)
out CurrentSystemName() ": A_ASSOCIATE.REQ [INFO]: apcon = " $apcon

$apcon varset asos_apcon
$demand varset asos_demand

unbufferit namer saved_address 1

P_CONNECT.REQ eventdown address $saved_address quality $quality demand  $demand context $context
return
```

## P_CONNECT.CONF
```
;параметры:  quality (буфер), context (буфер)
out CurrentSystemName() ": P_CONNECT.CONF [INFO]: apcon = " $asos_apcon

$asos_apcon == "GS" if handle_gs_conf
$asos_apcon == "DT" if handle_dt_conf
return


handle_gs_conf:
	; после установления GS-ассоциации отправляем запрос в справочник
	out CurrentSystemName() ": P_CONNECT.CONF [INFO]: GS established, sending name to Guide"

	buffer_tmp bufferit 1+sizeof(name_to_resolve) 3 1 $name_to_resolve sizeof(name_to_resolve)

	A_DATA.REQ timer t_gs_data 0 userdata $buffer_tmp
	return

handle_dt_conf:
	; DT-ассоциация установлена — подтверждаем инициацию передачи
	out CurrentSystemName() ": P_CONNECT.CONF [INFO]: DT established, confirming transfer init"

	A_TRANSFER_INIT.CONF generateup context $curr_context quality $curr_quality
	return
```

## A_DATA.REQ
```
;параметры: userdata(буфер)
break
out CurrentSystemName() ": A_DATA.REQ [INFO]: enqueue data"

qcount(data_queue) != 0 if skip
P_PLEASE_TOKENS.REQ eventdown token 1

skip:
	queue data_queue $userdata
	return
```

## A_RESOLVE.IND.DT
```

unbufferit buffer_with_address guide_address 1 guide_char_1 1 guide_char_2 1

out CurrentSystemName() ": A_RESOLVE.IND.DT [INFO]: address = " $guide_address

receiver_buffer bufferit 1+sizeof(curr_system_name) $guide_address 1 $curr_system_name sizeof(curr_system_name)

A_ASSOCIATE.REQ timer t_ 100 namer $receiver_buffer namei $initiator_buffer quality $curr_quality demand $gs_demand context $curr_context apcon "DT"
return
```

## P_CONNECT.IND
```
;параметры:  address (число), quality (буфер), demand (буфер)
out CurrentSystemName() ": P_CONNECT.IND [INFO]: incoming connect, start expedited handshake"

$address varset saved_address
$quality varset curr_quality
$demand  varset asos_demand

; expedited пакет: [PACK_TYPE_EXP][CON_REQ_TYPE]
buffer_tmp bufferit 2 $PACK_TYPE_EXP 1 $CON_REQ_TYPE 1

P_EXPEDITED_DATA.REQ eventdown userdata $buffer_tmp
return
```

## P_EXPEDITED_DATA.IND
```
;параметры: userdata(буфер)
unbufferit userdata pack_type 1 d_type 1 buffer_tmp sizeof(userdata)-2

out CurrentSystemName() ": P_EXPEDITED_DATA.IND [INFO]: pack_type = " $pack_type " d_type = " $d_type

;$pack_type != $PACK_TYPE_EXP if skip ???
$d_type == $CON_REQ_TYPE if handle_con_req
$d_type == $CON_RES_TYPE if handle_con_res
return

handle_con_req:
	out CurrentSystemName() ": P_EXPEDITED_DATA.IND [INFO]: got CON_REQ, send CON_RES apcon=" $asos_apcon

	; пакет: [PACK_TYPE_EXP][CON_RES_TYPE][asos_apcon...]
	buffer_send bufferit 2+sizeof(asos_apcon) $PACK_TYPE_EXP 1 $CON_RES_TYPE 1 $asos_apcon sizeof(asos_apcon)

	P_EXPEDITED_DATA.REQ eventdown userdata $buffer_send
	return

handle_con_res:
	out CurrentSystemName() ": P_EXPEDITED_DATA.IND [INFO]: got CON_RES, parse apcon"

	unbufferit buffer_tmp asos_apcon sizeof(buffer_tmp)
	out CurrentSystemName() ": P_EXPEDITED_DATA.IND [INFO]: remote apcon = " $asos_apcon

	$asos_apcon == "DT" if handle_dt
	return

handle_dt:
	out CurrentSystemName() ": P_EXPEDITED_DATA.IND [INFO]: DT requested -> send A_TRANSFER_INIT.IND"

	; получим имя инициатора по адресу
	locguide($saved_address) varset initiator_name

	A_TRANSFER_INIT.IND generateup namei $initiator_name quality $curr_quality
	return
```

## A_TRANSFER_INIT.RESP
```
;параметры: namei (строка), context (буфер), quality (буфер)
out CurrentSystemName() ": A_TRANSFER_INIT.RESP [INFO]: accept transfer, start DT associate response"

; сформировать буфер namei: [addr][namei]
initiator_buffer bufferit 1+sizeof(namei) $saved_address 1 $namei sizeof(namei)

A_ASSOCIATE.RESP timer t_ 0 namei $initiator_buffer context $context quality $quality apcon "DT"
return
```

## A_ASSOCIATE.RESP
```
;параметры: namei (буфер), context (буфер), quality (буфер), apcon (строка)
out CurrentSystemName() ": A_ASSOCIATE.RESP [INFO]: apcon = " $apcon ", down P_CONNECT.RESP"

$apcon varset asos_apcon
P_CONNECT.RESP eventdown address $saved_address quality $quality context $context
return
```

## P_DATA.IND
```
;параметры: userdata (буфер)
out CurrentSystemName() ": P_DATA.IND [INFO]: apcon=" $asos_apcon

$asos_apcon == "DT" if handle_dt
$asos_apcon == "GS" if handle_gs
return

handle_dt:
	A_DATA.IND generateup userdata $userdata
	return

handle_gs:
	; формат: [d_type][guide_id][guide_address][time1][time2][char1][char2]
	unbufferit userdata data_type 1 guide_id 1 guide_address 1 guide_time_1 1 guide_time_2 1 guide_char_1 1 guide_char_2 1

	out CurrentSystemName() ": P_DATA.IND [INFO]: Guide answer id=" $guide_id " addr=" $guide_address

	$guide_id == $GOT_ADDRESS if gs_got
	$guide_id == $ADDRESS_AFTER_TIME_1 if gs_wait
	$guide_id == $ADDRESS_BETWEEN_TIME_1_AND_2 if gs_wait
	$guide_id == $NO_ADDRESS if gs_fail
	$guide_id == $UNKNOWN_NAME_MAY_BE_REPLACE if gs_fail
	return

gs_got:
	$guide_address varset saved_address
	; сохранить буфер для отдачи в DT после release GS
	guide_buffer bufferit 4 $guide_address 1 0 1 " " 1 " " 1
	; закроем GS, а в P_RELEASE.CONF(GS) перекинем в A_RESOLVE.IND.DT
	P_RELEASE.REQ eventdown
	return

gs_wait:
	out CurrentSystemName() ": P_DATA.IND [INFO]: Guide says wait, retry after " $guide_time_1

	buffer_tmp bufferit 1+sizeof(name_to_resolve) 3 1 $name_to_resolve sizeof(name_to_resolve)
	A_DATA.REQ timer t_gs_retry $guide_time_1 userdata $buffer_tmp
	return

gs_fail:
	out CurrentSystemName() ": P_DATA.IND [WARN]: Guide failed to resolve name=" $name_to_resolve
	P_RELEASE.REQ eventdown
	return
```

## P_GIVE_TOKENS.IND
```
;параметры:  token (число)
out CurrentSystemName() ": P_GIVE_TOKENS.IND [INFO]: token=" $token

$token == 1 if send_data
$token == 2 if do_sync
return

send_data:
	qcount(data_queue) == 0 if empty
	P_DATA.REQ eventdown userdata dequeue(data_queue)
	goto send_data

do_sync:
	P_SYNC_MAJOR.REQ eventdown
	return

empty:
	P_PLEASE_TOKENS.REQ eventdown token 2
	return
```

## P_PLEASE_TOKENS.IND
```
;параметры:  token (число)
out CurrentSystemName() ": P_PLEASE_TOKENS.IND [INFO]: token = " $token

P_GIVE_TOKENS.REQ eventdown token $token

; если token = 1 — сброс очереди
$token != 1 if skip
clearqueue data_queue

skip:
	return
```

## P_P_EXCEPTION.IND
```
;параметры:  error (число)
out CurrentSystemName() ": P_P_EXCEPTION.IND [WARN]: error = " $error

$error == 1 if need_data_t
$error == 2 if need_sync_t
return

need_data_t:
	qcount(data_queue) == 0 if break
	P_DATA.REQ eventdown userdata dequeue(data_queue)
	goto need_data_t

need_sync_t:
	P_SYNC_MAJOR.REQ eventdown
	return

break:
	P_PLEASE_TOKENS.REQ eventdown token 2
	return
```

## P_SYNC_MAJOR.IND
```
;параметры:  нет
out CurrentSystemName() ": P_SYNC_MAJOR.IND [INFO]: got sync indication, down P_SYNC_MAJOR.RESP"
P_SYNC_MAJOR.RESP eventdown
return
```

## P_SYNC_MAJOR.CONF
```
;параметры:  нет
out CurrentSystemName() ": P_SYNC_MAJOR.CONF [INFO]: sync confirmed"
return
```

## P_RELEASE.IND
```
;параметры:  нет
out CurrentSystemName() ": P_RELEASE.IND [INFO]: apcon = " $asos_apcon
$asos_apcon == "DT" if handle_dt
return

handle_dt:
	A_TERMINATE.IND generateup
	return
```

## A_RELEASE.RESP
```
;параметры: apcon (строка)
out CurrentSystemName() ": A_RELEASE.RESP [INFO]: apcon = " $apcon ", down P_RELEASE.RESP"

$apcon varset asos_apcon
P_RELEASE.RESP eventdown
return
```

## A_TERMINATE.RESP
```
; параметры: нет
; Логика: TERMINATE.RESP -> RELEASE.RESP для текущей ассоциации (обычно DT)
out CurrentSystemName() ": A_TERMINATE.RESP [INFO]: terminate resp -> release resp apcon=" $asos_apcon
A_RELEASE.RESP timer t_term_resp 0 apcon $asos_apcon
return
```

## A_SYNC_MAJOR.REQ
```
; параметры: нет
out CurrentSystemName() ": A_SYNC_MAJOR.REQ [INFO]: down P_SYNC_MAJOR.REQ"
P_SYNC_MAJOR.REQ eventdown
return
```

## P_EXPEDITED_DATA.REQ.WAIT
```
; параметры: userdata (буфер)
; Используется как повторная отправка expedited (например, если внизу попросили подождать/повторить)
out CurrentSystemName() ": P_EXPEDITED_DATA.REQ.WAIT [INFO]: resend expedited"
P_EXPEDITED_DATA.REQ eventdown userdata $userdata
return
```

## 
```

```

## 
```

```

## 
```

```

## 
```

```



# Уровень представления
## P_INIT.REQ
```
; Полезные константы
empty_buffer declare buffer
custom_splitter declare string
"|" varset custom_splitter


; Глобальные переменные
context_buffer declare buffer
curr_address declare integer
quality_buffer declare buffer
curr_syntax declare integer

open_splitter declare string
close_splitter declare string
tr_syntax_1 declare integer
tr_syntax_2 declare integer

; ENUM типов сообщений
CONNECT_REQ declare integer
0 varset CONNECT_REQ

CONNECT_RESP declare integer
1 varset CONNECT_RESP

EXP_DATA declare integer
2 varset EXP_DATA

; Передача данных (внутренняя)
msg_type declare integer
msg_data declare buffer
payload declare buffer
payload_len declare integer

; Передача данных (DATA.REQ)
msg_id declare integer
str_data declare string
word declare string
word_len declare integer

; ENUM идентификаторов данных в DATA.REQ
STRUCT declare integer
1 varset STRUCT

NUMBER declare integer
2 varset NUMBER

STRING declare integer
3 varset STRING

BUFFER declare integer
4 varset BUFFER

; ENUM id синтаксиса передачи
CODE_BY_LENGTH declare integer
1 varset CODE_BY_LENGTH

CODE_BY_END declare integer
2 varset CODE_BY_END

0 varset payload_len
```

## P_CONNECT.REQ
```
;параметры:  address (число), quality (буфер), demand (буфер), context (буфер)
out CurrentSystemName() ": P_CONNECT.REQ [INFO]: sending S_CONNECT.REQ"

out CurrentSystemName() ": P_CONNECT.REQ [DEBUG]: address = " $address ", size of context = " sizeof(context)
$context varset context_buffer
$address varset curr_address

unbufferit context open_splitter 1 close_splitter 1
out CurrentSystemName() ": P_CONNECT.REQ [INFO]: open_split = " $open_splitter ", close_split = " $close_splitter

S_CONNECT.REQ eventdown address $address quality $quality demand $demand

return
```

## P_CONNECT.RESP
```
;параметры:  address (число), quality (буфер), context (буфер)
out CurrentSystemName() ": P_CONNECT.RESP [INFO]: got context with size = " sizeof(context) " from address = " $address

$address varset curr_address
$context varset context_buffer

out CurrentSystemName() ": P_CONNECT.RESP [INFO]: down S_CONNECT.RESP"

S_CONNECT.RESP eventdown address $address quality $quality

return
```

## P_U_ABORT.REQ
```
;параметры:  нет
out CurrentSystemName() ": P_U_ABORT.REQ [INFO]: down S_U_ABORT.REQ"
S_U_ABORT.REQ eventdown
return
```

## S_CONNECT.CONF
```
;параметры:  quality (буфер)
$quality varset quality_buffer
out CurrentSystemName() ": S_CONNECT.CONF [INFO]: down S_EXPEDITED_DATA.REQ with context"

msg_data bufferit 1+sizeof(context_buffer) $CONNECT_REQ 1 $context_buffer sizeof(context_buffer)

S_EXPEDITED_DATA.REQ eventdown userdata $msg_data

return
```

## S_CONNECT.IND
```
;параметры:  address (число), quality (буфер), demand (буфер)
out CurrentSystemName() ": S_CONNECT.IND [INFO]: up P_CONNECT.IND"
P_CONNECT.IND generateup address $address quality $quality demand $demand
```

## S_EXPEDITED_DATA.IND
```
;параметры:  userdata (буфер)
out CurrentSystemName() ": S_EXPEDITED_DATA.IND [INFO]: get data with size = " sizeof(userdata)

unbufferit userdata msg_type 1 payload sizeof(userdata)-1

$msg_type == $CONNECT_REQ 	if handle_connect_req
$msg_type == $CONNECT_RESP 	if handle_connect_resp
$msg_type == $EXP_DATA 		if handle_exp_data

out CurrentSystemName() + ": S_EXPEDITED_DATA.IND [ERROR]: unknown presentation message type = " $msg_type
return

handle_connect_req:
	; payload = [open_splitter][close_splitter][syntax_1][syntax_2?]
	sizeof(payload) == 4 if req_unpack_2
	sizeof(payload) == 3 if req_unpack_1

	out CurrentSystemName() + ": S_EXPEDITED_DATA.IND [ERROR]: wrong format of payload: size = " sizeof(payload)
	return

req_unpack_1:
	unbufferit payload open_splitter 1 close_splitter 1 tr_syntax_1 1
	out CurrentSystemName() ": S_EXPEDITED_DATA.IND [INFO]: unpack one: os = " $open_splitter " cs = " $close_splitter
	out CurrentSystemName() ": S_EXPEDITED_DATA.IND [INFO]: unpack one: s1 = " $tr_syntax_1 " s2 is null"
	goto send_resp

req_unpack_2:
	unbufferit payload open_splitter 1 close_splitter 1 tr_syntax_1 1 tr_syntax_2 1
	out CurrentSystemName() ": S_EXPEDITED_DATA.IND [INFO]: unpack two: os = " $open_splitter " cs = " $close_splitter
	out CurrentSystemName() ": S_EXPEDITED_DATA.IND [INFO]: unpack two: s1 = " $tr_syntax_1 " s2 = " $tr_syntax_2
	goto send_resp

send_resp:
	$tr_syntax_1 varset curr_syntax
	out CurrentSystemName() ": S_EXPEDITED_DATA.IND [INFO]: send resp through S_EXPEDITED_DATA.REQ"
	msg_data bufferit 1+sizeof(payload) $CONNECT_RESP 1 $payload sizeof(payload)
	S_EXPEDITED_DATA.REQ eventdown userdata $msg_data
	return

handle_connect_resp:
	; payload = [open_splitter][close_splitter][syntax_1][syntax_2?]
	sizeof(payload) == 4 if resp_unpack_2
	sizeof(payload) == 3 if resp_unpack_1
	
	out CurrentSystemName() + ": S_EXPEDITED_DATA.IND [ERROR]: wrong format of payload: size = " sizeof(payload)
	return

resp_unpack_1:
	unbufferit payload open_splitter 1 close_splitter 1 tr_syntax_1 1
	out CurrentSystemName() ": S_EXPEDITED_DATA.IND [INFO]: unpack one: os = " $open_splitter " cs = " $close_splitter
	out CurrentSystemName() ": S_EXPEDITED_DATA.IND [INFO]: unpack one: s1 = " $tr_syntax_1 " s2 is null"
	goto send_conf

resp_unpack_2:
	unbufferit payload open_splitter 1 close_splitter 1 tr_syntax_1 1 tr_syntax_2 1
	out CurrentSystemName() ": S_EXPEDITED_DATA.IND [INFO]: unpack two: os = " $open_splitter " cs = " $close_splitter
	out CurrentSystemName() ": S_EXPEDITED_DATA.IND [INFO]: unpack two: s1 = " $tr_syntax_1 " s2 = " $tr_syntax_2
	goto send_conf

send_conf:
	$tr_syntax_1 varset curr_syntax
	out CurrentSystemName() ": S_EXPEDITED_DATA.IND [INFO]: send conf through S_EXPEDITED_DATA.REQ"
	payload bufferit 3 $open_splitter 1 $close_splitter 1 $tr_syntax_1 1
	P_CONNECT.CONF generateup quality $quality_buffer context $payload 
	return

handle_exp_data:
	; payload = [inner_msg_type][...]
	out CurrentSystemName() ": S_EXPEDITED_DATA.IND [INFO]: handle EXP_DATA, size = " sizeof(payload)
	
	; Сохраним буффер на случай не структуры
	$payload varset msg_data

	; inner message = обычный presentation-data
	unbufferit payload msg_type 1 payload sizeof(payload)-1
	out CurrentSystemName() ": S_EXPEDITED_DATA.IND [INFO]: inner msg_type = " $msg_type

	; ---------- НЕ STRUCT ----------
	$msg_type != $STRUCT if exp_not_struct

	unbufferit payload curr_syntax 1 payload sizeof(payload)-1
	out CurrentSystemName() ": S_EXPEDITED_DATA.IND [INFO]: EXP STRUCT syntax = " $curr_syntax ", payload size = " sizeof(payload)

	$curr_syntax == $CODE_BY_END    if exp_decode_end
	$curr_syntax == $CODE_BY_LENGTH if exp_decode_len

	out CurrentSystemName() ": S_EXPEDITED_DATA.IND [ERROR]: unknown EXP STRUCT syntax = " $curr_syntax
	P_EXPEDITED_DATA.IND generateup userdata $userdata
	return


exp_not_struct:
	; expedited, но без presentation-кодирования
	P_EXPEDITED_DATA.IND generateup userdata $msg_data
	return

exp_decode_end:
	unbufferit payload str_data sizeof(payload)
	out CurrentSystemName() ": S_EXPEDITED_DATA.IND [INFO]: EXP decode_end raw = " $str_data

	$empty_buffer varset msg_data
	msg_data bufferit sizeof(msg_data)+sizeof(open_splitter) $msg_data sizeof(msg_data) $open_splitter sizeof(open_splitter)

	goto exp_end_loop


exp_end_loop:
	sizeof(str_data) == 0 if exp_end_finish

	pos($custom_splitter, str_data) == 0 if exp_end_last

	copy(str_data, 1, pos($custom_splitter, str_data)-1) varset word
	delete str_data 1 sizeof(word)+sizeof(custom_splitter)

	msg_data bufferit sizeof(msg_data)+sizeof(open_splitter)+sizeof(word)+sizeof(close_splitter) $msg_data sizeof(msg_data) $open_splitter sizeof(open_splitter) $word sizeof(word) $close_splitter sizeof(close_splitter)

	goto exp_end_loop


exp_end_last:
	$str_data varset word
	delete str_data 1 sizeof(word)

	msg_data bufferit sizeof(msg_data)+sizeof(open_splitter)+sizeof(word)+sizeof(close_splitter) $msg_data sizeof(msg_data) $open_splitter sizeof(open_splitter) $word sizeof(word) $close_splitter sizeof(close_splitter)

	goto exp_end_finish


exp_end_finish:
	msg_data bufferit sizeof(msg_data)+sizeof(close_splitter) $msg_data sizeof(msg_data) $close_splitter sizeof(close_splitter)

	payload bufferit 1+sizeof(msg_data) $STRUCT 1 $msg_data sizeof(msg_data)
	P_EXPEDITED_DATA.IND generateup userdata $payload
	return

exp_decode_len:
	$empty_buffer varset msg_data
	msg_data bufferit sizeof(msg_data)+sizeof(open_splitter) $msg_data sizeof(msg_data) $open_splitter sizeof(open_splitter)

	goto exp_len_loop


exp_len_loop:
	sizeof(payload) == 0 if exp_len_finish

	unbufferit payload word_len 1 payload sizeof(payload)-1

	unbufferit payload word $word_len payload sizeof(payload)-$word_len
	CurrentSystemName() ": S_EXPEDITED_DATA.IND [INFO]: decode word = " $word

	msg_data bufferit sizeof(msg_data)+sizeof(open_splitter)+sizeof(word)+sizeof(close_splitter) $msg_data sizeof(msg_data) $open_splitter sizeof(open_splitter) $word sizeof(word) $close_splitter sizeof(close_splitter)

	goto exp_len_loop


exp_len_finish:
	msg_data bufferit sizeof(msg_data)+sizeof(close_splitter) $msg_data sizeof(msg_data) $close_splitter sizeof(close_splitter)

	payload bufferit 1+sizeof(msg_data) $STRUCT 1 $msg_data sizeof(msg_data)
	P_EXPEDITED_DATA.IND generateup userdata $payload
	return
```


## S_U_ABORT.IND
```
;параметры:  нет
out CurrentSystemName() ": S_U_ABORT.IND [INFO]: up P_U_ABORT.IND"
P_U_ABORT.IND generateup
return
```

## S_P_ABORT.IND
```
;параметры:  нет
out CurrentSystemName() ": S_P_ABORT.IND [INFO]: up P_P_ABORT.IND"
P_P_ABORT.IND generateup
return
```

## P_DATA.REQ
```
;параметры:  userdata (буфер)
unbufferit userdata msg_type 1 payload sizeof(userdata)-1
out CurrentSystemName() ": P_DATA.REQ [INFO]: got data with id = " $msg_type ", syntax = " $curr_syntax

$msg_type == $STRUCT if handle_struct
S_DATA.REQ eventdown userdata $userdata
return

handle_struct:
	unbufferit payload str_data sizeof(payload)

	$curr_syntax == $CODE_BY_LENGTH 	if len_type
	$curr_syntax == $CODE_BY_END 		if end_type

	out CurrentSystemName() ": P_DATA.REQ [ERROR]: unknown syntax = " $curr_syntax
	return

end_type:
	$empty_buffer varset payload
	delete str_data 1 1 ; убираем символ начала структуры
	out CurrentSystemName() ": P_DATA.REQ [INFO]: str_data after delete = " $str_data

	goto end_while

end_while:
	sizeof(str_data) == 1 if send_data
	; Убираем открывающий символ
	delete str_data 1 1

	; Берем все до закрывающего символа
	out $str_data
	out pos($close_splitter, str_data)
	copy(str_data, 1, pos($close_splitter, str_data) - 1) varset word 
	out CurrentSystemName() ": P_DATA.REQ [INFO]: get word = " $word

	; очищаем слово и закрывающий символ
	delete str_data 1 sizeof(word)+1

	; Упаковываем разделитель + слово 
	payload bufferit sizeof(payload)+sizeof(custom_splitter)+sizeof(word) $payload sizeof(payload) $word sizeof(word) $custom_splitter sizeof(custom_splitter)

	goto end_while

return

len_type:
	; Кодирование: [len(1 byte)][word bytes]...
	$empty_buffer varset payload

	; Убираем символ начала структуры (open_splitter)
	delete str_data 1 1
	out CurrentSystemName() ": P_DATA.REQ [INFO]: len_type: str_data after delete = " $str_data

	goto len_while

len_while:
	; Если остался только финальный close_splitter — заканчиваем и отправляем
	sizeof(str_data) == 1 if send_data

	; Убираем открывающий символ перед очередным словом
	delete str_data 1 1

	; Находим слово до закрывающего символа
	copy(str_data, 1, pos($close_splitter, str_data) - 1) varset word
	out CurrentSystemName() ": P_DATA.REQ [INFO]: len_type: get word = " $word

	; Пусть длина слова = 1 (надеемся)
	sizeof(word) varset word_len

	; Пакуем: len + word
	payload bufferit sizeof(payload)+1+sizeof(word) $payload sizeof(payload) $word_len 1 $word sizeof(word)

	; Удаляем слово + закрывающий символ
	delete str_data 1 sizeof(word)+1

	goto len_while

send_data:
	msg_data bufferit 2+sizeof(payload) $msg_type 1 $curr_syntax 1 $payload sizeof(payload)

	S_DATA.REQ eventdown userdata $msg_data
	return
```

## P_SYNC_MAJOR.REQ
```
;параметры:  нет
out CurrentSystemName() + ": P_SYNC_MAJOR.REQ [INFO]: down S_SYNC_MAJOR.REQ"
S_SYNC_MAJOR.REQ eventdown
return
```

## P_SYNC_MAJOR.RESP
```
;параметры:  нет
out CurrentSystemName() + ": P_SYNC_MAJOR.RESP [INFO]: down S_SYNC_MAJOR.RESP"
S_SYNC_MAJOR.RESP eventdown
```

## S_SYNC_MAJOR.IND
```
;параметры:  нет
out CurrentSystemName() + ": S_SYNC_MAJOR.IND [INFO]: up P_SYNC_MAJOR.IND"
P_SYNC_MAJOR.IND generateup
```

## S_DATA.IND
```
;параметры:  userdata (буфер)
unbufferit userdata msg_type 1 payload sizeof(userdata)-1
out CurrentSystemName() ": P_DATA.REQ [INFO]: got data with id = " $msg_type ", size = " sizeof(userdata)

$msg_type == $STRUCT if handle_struct
P_DATA.IND generateup userdata $userdata
return

handle_struct:
	unbufferit payload curr_syntax 1 payload sizeof(payload)-1
	out CurrentSystemName() ": S_DATA.IND [INFO]: STRUCT syntax = " $curr_syntax ", enc_payload_size = " sizeof(payload)

	$curr_syntax == $CODE_BY_END 	if decode_end
	$curr_syntax == $CODE_BY_LENGTH if decode_len

	out CurrentSystemName() ": S_DATA.IND [ERROR]: unknown syntax = " $curr_syntax
	return

decode_end:
	unbufferit payload str_data sizeof(payload)
	out CurrentSystemName() ": S_DATA.IND [INFO]: decode_end raw = " $str_data

	$empty_buffer varset msg_data

	; внешний open
	msg_data bufferit sizeof(msg_data)+sizeof(open_splitter) $msg_data sizeof(msg_data) $open_splitter sizeof(open_splitter)

	goto end_loop


end_loop:
	; пустая строка — закончили
	sizeof(str_data) == 0 if end_finish

	; если разделитель не найден — последнее слово
	pos($custom_splitter, str_data) == 0 if end_last_word

	; слово до разделителя
	copy(str_data, 1, pos($custom_splitter, str_data)-1) varset word

	out CurrentSystemName() ": S_DATA.IND [INFO]: got word " $word
	; удаляем word + splitter
	delete str_data 1 sizeof(word)+sizeof(custom_splitter)

	; добавляем "{word}"
	msg_data bufferit sizeof(msg_data)+sizeof(open_splitter)+sizeof(word)+sizeof(close_splitter) $msg_data sizeof(msg_data) $open_splitter sizeof(open_splitter) $word sizeof(word) $close_splitter sizeof(close_splitter)

	goto end_loop

end_last_word:
	; остаток — это слово
	$str_data varset word
	; очищаем строку
	delete str_data 1 sizeof(word)

	; добавляем "{word}"
	msg_data bufferit sizeof(msg_data)+sizeof(open_splitter)+sizeof(word)+sizeof(close_splitter) $msg_data sizeof(msg_data) $open_splitter sizeof(open_splitter) $word sizeof(word) $close_splitter sizeof(close_splitter)

	goto end_finish


end_finish:
	; внешний close
	msg_data bufferit sizeof(msg_data)+sizeof(close_splitter) $msg_data sizeof(msg_data) $close_splitter sizeof(close_splitter)

	; поднимаем наверх как [STRUCT][структура-строка]
	payload bufferit 1+sizeof(msg_data) $msg_type 1 $msg_data sizeof(msg_data)
	P_DATA.IND generateup userdata $payload
	return

decode_len:
	; выходной буфер
	$empty_buffer varset msg_data

	; внешний open
	msg_data bufferit sizeof(msg_data)+sizeof(open_splitter) $msg_data sizeof(msg_data) $open_splitter sizeof(open_splitter)

	goto len_loop


len_loop:
	; если ничего не осталось — финализируем
	sizeof(payload) == 0 if len_finish

	; читаем длину
	unbufferit payload word_len 1 payload sizeof(payload)-1

	; читаем слово длиной word_len
	unbufferit payload word $word_len payload sizeof(payload)-$word_len
	out CurrentSystemName() ": P_DATA.REQ [INFO]: decode word = " $word

	; добавляем "{word}"
	msg_data bufferit sizeof(msg_data)+sizeof(open_splitter)+sizeof(word)+sizeof(close_splitter) $msg_data sizeof(msg_data) $open_splitter sizeof(open_splitter) $word sizeof(word) $close_splitter sizeof(close_splitter)

	goto len_loop


len_finish:
	; внешний close
	msg_data bufferit sizeof(msg_data)+sizeof(close_splitter) $msg_data sizeof(msg_data) $close_splitter sizeof(close_splitter)

	; наверх: [STRUCT][структура]
	payload bufferit 1+sizeof(msg_data) $msg_type 1 $msg_data sizeof(msg_data)
	P_DATA.IND generateup userdata $payload
	return
```

## P_PLEASE_TOKENS.REQ
```
;параметры:  token (число)
out CurrentSystemName() ": P_PLEASE_TOKENS.REQ [INFO]: down S_PLEASE_TOKENS.REQ with token"
S_PLEASE_TOKENS.REQ eventdown token $token
```

## S_SYNC_MAJOR.CONF
```
;параметры:  нет
out CurrentSystemName() ": S_SYNC_MAJOR.CONF [INFO]: up P_SYNC_MAJOR.CONF"
P_SYNC_MAJOR.CONF generateup
```

## S_PLEASE_TOKENS.IND
```
;параметры:  token (число)
out CurrentSystemName() + ": S_PLEASE_TOKENS.IND up P_PLEASE_TOKENS.IND"
P_PLEASE_TOKENS.IND generateup token $token
```

## P_GIVE_TOKENS.REQ
```
;параметры:  token (число)
out CurrentSystemName() ": P_GIVE_TOKENS.REQ down S_GIVE_TOKENS.REQ"
S_GIVE_TOKENS.REQ eventdown token $token
```

## S_GIVE_TOKENS.IND
```
;параметры:  token (число)
out CurrentSystemName() ": S_GIVE_TOKENS.IND up P_GIVE_TOKENS.IND"
P_GIVE_TOKENS.IND generateup token $token
```

## S_P_EXCEPTION.IND
```
;параметры:  error (число)
out CurrentSystemName() + ": S_P_EXCEPTION.IND up P_P_EXCEPTION.IND with error = " $error
P_P_EXCEPTION.IND generateup error $error
```

## P_EXPEDITED_DATA.REQ
```
;параметры:  userdata (буфер)
unbufferit userdata msg_type 1 payload sizeof(userdata)-1
out CurrentSystemName() ": P_EXPEDITED_DATA.REQ [INFO]: got data id = " $msg_type ", syntax = " $curr_syntax

$msg_type == $STRUCT if handle_struct

msg_data bufferit 1+sizeof(userdata) $EXP_DATA 1 $userdata sizeof(userdata)
S_EXPEDITED_DATA.REQ eventdown userdata $msg_data
return

handle_struct:
	; payload -> строка структуры
	unbufferit payload str_data sizeof(payload)

	$curr_syntax == $CODE_BY_LENGTH if len_type
	$curr_syntax == $CODE_BY_END 	if end_type

	out CurrentSystemName() ": P_EXPEDITED_DATA.REQ [ERROR]: unknown syntax = " $curr_syntax
	return


; ------------------------------------------------------------
; CODE_BY_END: payload = word1|word2|...
; ------------------------------------------------------------
end_type:
	$empty_buffer varset payload
	delete str_data 1 1 ; убираем внешний open структуры

	goto end_while

end_while:
	sizeof(str_data) == 1 if send_data

	; убрать внутренний open перед словом
	delete str_data 1 1

	; слово до close_splitter
	copy(str_data, 1, pos($close_splitter, str_data) - 1) varset word
	out CurrentSystemName() ": P_EXPEDITED_DATA.REQ [INFO]: end_type word = " $word

	; удалить word + close_splitter
	delete str_data 1 sizeof(word)+1

	; payload += word + '|'
	payload bufferit sizeof(payload)+sizeof(word)+sizeof(custom_splitter) \
		$payload sizeof(payload) $word sizeof(word) $custom_splitter sizeof(custom_splitter)

	goto end_while


; ------------------------------------------------------------
; CODE_BY_LENGTH: payload = [len][word][len][word]...
; len = 1 байт
; ------------------------------------------------------------
len_type:
	$empty_buffer varset payload
	delete str_data 1 1 ; убираем внешний open структуры

	goto len_while

len_while:
	sizeof(str_data) == 1 if send_data

	; убрать внутренний open
	delete str_data 1 1

	; слово до close_splitter
	copy(str_data, 1, pos($close_splitter, str_data) - 1) varset word
	out CurrentSystemName() ": P_EXPEDITED_DATA.REQ [INFO]: len_type word = " $word

	; длина слова (1 байт)
	sizeof(word) varset msg_id

	; payload += [len][word]
	payload bufferit sizeof(payload)+1+sizeof(word) \
		$payload sizeof(payload) $msg_id 1 $word sizeof(word)

	; удалить word + close_splitter
	delete str_data 1 sizeof(word)+1

	goto len_while


; ------------------------------------------------------------
; Упаковка и отправка expedited
; ------------------------------------------------------------
send_data:
	; собираем внутреннее сообщение представления: [msg_type][curr_syntax][encoded_payload]
	msg_data bufferit 2+sizeof(payload) $msg_type 1 $curr_syntax 1 $payload sizeof(payload)

	; оборачиваем как expedited: [EXP_DATA][inner_msg]
	payload bufferit 1+sizeof(msg_data) $EXP_DATA 1 $msg_data sizeof(msg_data)

	S_EXPEDITED_DATA.REQ eventdown userdata $payload
	return

```

## P_RELEASE.REQ
```
;параметры:  нет
out CurrentSystemName() ": P_RELEASE.REQ [INFO]: down S_RELEASE.REQ"
S_RELEASE.REQ eventdown
```

## S_RELEASE.CONF
```
;параметры:  нет
out CurrentSystemName() ": S_RELEASE.CONF [INFO]: P_RELEASE.CONF"
P_RELEASE.CONF generateup
```

## S_RELEASE.IND
```
;параметры:  нет
out CurrentSystemName() ": S_RELEASE.IND [INFO]: P_RELEASE.IND"
P_RELEASE.IND generateup
```

## P_RELEASE.RESP
```
;параметры:  нет
out CurrentSystemName() ": P_RELEASE.RESP [INFO]: S_RELEASE.RESP"
S_RELEASE.RESP eventdown
```
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
    S_P_EXCEPTION.IND generateup error $no_data_marker_error
    return

give_token_no_sync:
    out CurrentSystemName() ": S_GIVE_TOKENS.REQ [ERROR]: no SYNC token to give"
    S_P_EXCEPTION.IND generateup error $no_sync_marker_error
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

(($token == 1) && ($has_data_marker == 1)) if send_ex_no_data_marker
(($token == 2) && ($has_sync_marker == 1)) if send_ex_no_sync_marker

; Формируем PDU: [MSG_PLEASE_TOKENS][token]
connection_params bufferit 2 $MSG_PLEASE_TOKENS 1 $token 1

out CurrentSystemName() ": S_PLEASE_TOKENS.REQ [INFO]: sending PLEASE_TOKENS via T_DATA.REQ"
T_DATA.REQ eventdown userdata $connection_params

return

send_ex_no_data_marker:
	out CurrentSystemName() ": S_GIVE_TOKENS.REQ [ERROR]: no DATA token to give"
    S_P_EXCEPTION.IND generateup error $no_data_marker_error
    return

send_ex_no_sync_marker:
	out CurrentSystemName() ": S_GIVE_TOKENS.REQ [ERROR]: no SYNC token to give"
    S_P_EXCEPTION.IND generateup error $no_sync_marker_error
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