# Прикладной уровень

## A_INIT.REQ
```
; ---------- очереди ----------
data_queue declare queue

; ---------- контексты ----------
curr_context declare buffer
curr_quality declare buffer
curr_demand  declare buffer

; ---------- ассоциации ----------
asos_apcon   declare string
asos_demand  declare buffer

; ---------- адреса / имена ----------
saved_address        declare integer
name_to_resolve   declare string
cur_sys_name      declare string
resolved_namei    declare string

; ---------- временные буферы ----------
buffer_tmp        declare buffer
buffer_send       declare buffer
receiver_buffer      declare buffer
initiator_buffer      declare buffer
buffer_got_addr   declare buffer

; ---------- локгайд поля ----------
guide_id   declare integer
guide_address   declare integer
guide_time_1  declare integer
guide_time_2  declare integer
guide_char_1  declare integer
guide_char_2  declare integer

; ---------- expedited / service ----------
pack_type declare integer
d_type    declare integer

; ---------- ENUM ----------
CONNECT_REQ declare integer
1 varset CONNECT_REQ

CONNECT_RESP declare integer
2 varset CONNECT_RESP

PACK_EXP declare integer
3 varset PACK_EXP

t_term_req declare integer
t_term_resp declare integer
```

## A_TRANSFER_INIT.REQ
```
A_TRANSFER_INIT.REQ
out CurrentSystemName() ": A_TRANSFER_INIT.REQ [INFO]: namer=" $namer " namei=" $namei

$namer   varset _namer
$namei   varset _namei
$context varset curr_context
$quality varset curr_quality

A_RESOLVE.REQ timer t_resolve 0 name $_namer
return

```

## A_RESOLVE.REQ
```
out CurrentSystemName() ": A_RESOLVE.REQ [INFO]: name=" $name

$saved_address != 0 if RESOLVE_SAVED

locguide($name) varset buffer_tmp
unbufferit buffer_tmp guide_id 1 guide_address 1 guide_time_1 1 guide_time_2 1 guide_char_1 1 guide_char_2 1

out CurrentSystemName() ": A_RESOLVE.REQ [DEBUG]: type=" $guide_id " addr=" $guide_address

$guide_id == 0 if GOT_ADDR
($guide_id == 1) if WAIT_ADDR
($guide_id == 2) if WAIT_ADDR
($guide_id == 3) if NEED_GUIDE
($guide_id == 4) if NEED_GUIDE
return

GOT_ADDR:
$guide_address varset saved_address
buffer_tmp bufferit 4 $guide_address 1 0 1 " " 1 " " 1
A_RESOLVE.IND.DT timer t_resolved 0 buffer_with_address $buffer_tmp
return

WAIT_ADDR:
A_RESOLVE.REQ timer t_retry $guide_time_1 name $name
return

NEED_GUIDE:
$name varset name_to_resolve

locguide("Guide") varset buffer_tmp
unbufferit buffer_tmp guide_id 1 guide_address 1 guide_time_1 1 guide_time_2 1 guide_char_1 1 guide_char_2 1
receiver_buffer bufferit 6 $guide_address 1 "Guide" 5

CurrentSystemName() varset cur_sys_name
locguide($cur_sys_name) varset buffer_tmp
unbufferit buffer_tmp guide_id 1 guide_address 1 guide_time_1 1 guide_time_2 1 guide_char_1 1 guide_char_2 1
initiator_buffer bufferit 1+sizeof(cur_sys_name) $guide_address 1 $cur_sys_name sizeof(cur_sys_name)

A_ASSOCIATE.REQ timer t_gs 0 \
    namer $receiver_buffer \
    namei $initiator_buffer \
    quality $curr_quality \
    demand $gs_demand \
    context $gs_context \
    apcon "GS"
return

RESOLVE_SAVED:
buffer_tmp bufferit 4 $saved_address 1 0 1 " " 1 " " 1
A_RESOLVE.IND.DT timer t_saved 0 buffer_with_address $buffer_tmp
return

```

## A_RESOLVE.IND.DT
```
A_RESOLVE.IND.DT
unbufferit buffer_with_address guide_address 1 guide_char_1 1 guide_char_2 1

out CurrentSystemName() ": A_RESOLVE.IND.DT [INFO]: addr=" $guide_address

receiver_buffer bufferit 1+sizeof(cur_sys_name) $guide_address 1 $cur_sys_name sizeof(cur_sys_name)

A_ASSOCIATE.REQ timer t_dt 100 \
    namer $receiver_buffer \
    namei $initiator_buffer \
    quality $curr_quality \
    demand $gs_demand \
    context $curr_context \
    apcon "DT"
return

```

## A_ASSOCIATE.REQ
```
A_ASSOCIATE.REQ
out CurrentSystemName() ": A_ASSOCIATE.REQ [INFO]: apcon=" $apcon

$apcon varset asos_apcon
$demand varset asos_demand

unbufferit namer saved_address 1

P_CONNECT.REQ eventdown \
    address $saved_address \
    quality $quality \
    demand  $demand \
    context $context
return

```

## P_CONNECT.IND
```
P_CONNECT.IND
out CurrentSystemName() ": P_CONNECT.IND [INFO]: expedited handshake"

$address varset saved_address
$quality varset curr_quality
$demand  varset curr_demand

buffer_tmp bufferit 2 $PACK_EXP 1 $CONNECT_REQ 1
P_EXPEDITED_DATA.REQ eventdown userdata $buffer_tmp
return

```

## P_EXPEDITED_DATA.IND
```
P_EXPEDITED_DATA.IND
unbufferit userdata pack_type 1 d_type 1 buffer_tmp sizeof(userdata)-2

out CurrentSystemName() ": P_EXPEDITED_DATA.IND [INFO]: d_type=" $d_type

$d_type == $CONNECT_REQ if CON_REQ
$d_type == $CONNECT_RESP if CON_RES
return

CON_REQ:
buffer_send bufferit 2+sizeof(asos_apcon) \
    $PACK_EXP 1 \
    $CONNECT_RESP  1 \
    $asos_apcon sizeof(asos_apcon)

P_EXPEDITED_DATA.REQ eventdown userdata $buffer_send
return

CON_RES:
unbufferit buffer_tmp asos_apcon sizeof(buffer_tmp)
$asos_apcon == "DT" if DT_OK
return

DT_OK:
locguide($saved_address) varset resolved_namei
A_TRANSFER_INIT.IND generateup namei $resolved_namei quality $curr_quality
return

```

## A_DATA.REQ
```
A_DATA.REQ
out CurrentSystemName() ": A_DATA.REQ [INFO]: enqueue data"

qcount(data_queue) != 0 if SKIP
P_PLEASE_TOKENS.REQ eventdown token 1
SKIP:
queue data_queue $userdata
return

```

## P_GIVE_TOKENS.IND
```
P_GIVE_TOKENS.IND
out CurrentSystemName() ": P_GIVE_TOKENS.IND [INFO]: token=" $token

$token == 1 if SEND_DATA
$token == 2 if DO_SYNC
return

SEND_DATA:
qcount(data_queue) == 0 if EMPTY
P_DATA.REQ eventdown userdata dequeue(data_queue)
goto SEND_DATA

DO_SYNC:
P_SYNC_MAJOR.REQ eventdown
return

EMPTY:
P_PLEASE_TOKENS.REQ eventdown token 2
return

```

## P_DATA.IND (DT)
```
P_DATA.IND
$asos_apcon == "DT" if DT_DATA
return

DT_DATA:
sendup A_DATA.IND userdata $userdata
return

```

## P_RELEASE.CONF
```
out CurrentSystemName() ": P_RELEASE.CONF [INFO]: apcon=" $asos_apcon
$asos_apcon == "DT" if DT
return
DT:
sendup A_TERMINATE.CONF
return
```

## P_RELEASE.IND
```
out CurrentSystemName() ": P_RELEASE.IND [INFO]: apcon=" $asos_apcon
$asos_apcon == "DT" if DT_IND
return
DT_IND:
sendup A_TERMINATE.IND
return

```

## A_TERMINATE.REQ
```
out CurrentSystemName() ": A_TERMINATE.REQ [INFO]: sending A_RELEASE.REQ (DT) to close transfer"
A_RELEASE.REQ timer t_term_req 0 apcon "DT"
return

```

## A_TERMINATE.RESP
```
out CurrentSystemName() ": A_TERMINATE.RESP [INFO]: sending A_RELEASE.RESP (DT) to confirm close transfer"
A_RELEASE.RESP timer t_term_resp 0 apcon "DT"
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

## 
```

```

## 
```

```

## 
```

```

