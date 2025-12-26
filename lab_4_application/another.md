# Прикладной уровень

## A_INIT.REQ
```
integer declare saved_addr
0 setto saved_addr
integer declare lg_type
integer declare lg_addr
integer declare lg_time1
integer declare lg_time2
integer declare lg_char1
integer declare lg_char2
integer declare a_namer
integer declare a_namei
integer declare pack_type
integer declare d_type
integer declare need_data_t
integer declare need_sync_t

integer declare con_req_type
integer declare con_res_type
1 setto con_req_type
2 setto con_res_type

buffer declare _context
buffer declare _quality
buffer declare _demand
buffer declare tmp_buf
buffer declare namei_buf
buffer declare namer_buf
buffer declare gs_quality
gs_quality pack 1 1 1 1 2
buffer declare gs_demand
gs_demand pack 1 1 1
buffer declare gs_context
gs_context pack "{" 1 "}" 1 2 1 2 1 4
buffer declare asos_demand
buffer declare got_addr_buf
buffer declare send_buf
buffer declare last_data


string declare _namer
string declare _namei
string declare cur_sys_name
string declare name_to_resolve
string declare asos_apcon
string declare n_namer
string declare n_namei
string declare tmp_str


integer declare func_timer

queue declare data_q
```

## A_ASSOCIATE.REQ
```
;параметры: namer (буфер), namei (буфер), quality (буфер), demand (буфер), context (буфер), apcon (строка)
namer unbufferit a_namer 1 n_namer sizeof(namer) - 1
namei unbufferit a_namei 1 n_namei sizeof(namei) - 1
out CurrentSystemName() ": A_ASSOCIATE.REQ a_namer = " $a_namer " n_namer = " $n_namer " a_namei = " $a_namei " n_namei = " $n_namei " apcon = " $apcon
$apcon setto asos_apcon
$demand setto asos_demand
address $a_namer quality $quality demand $demand context $context eventdown P_CONNECT.REQ

```

## A_ASSOCIATE.RESP
```
;параметры: namei (буфер), context (буфер), quality (буфер), apcon (строка)
out CurrentSystemName() ": A_ASSOCIATE.RESP apcon = " $apcon
address $saved_addr quality $_quality context $context eventdown P_CONNECT.RESP
```

## A_DATA.REQ
```
;параметры: userdata(буфер)
out CurrentSystemName() ": A_DATA.REQ "
;$userdata setto last_data
qcount(data_q) != 0 if skip
token 1 eventdown P_PLEASE_TOKENS.REQ
skip:
queue $userdata data_q


;userdata $userdata eventdown P_DATA.REQ
;timeevent A_SYNC_MAJOR.REQ func_timer 3

```

## A_RELEASE.REQ
```
;параметры: apcon (строка)
out CurrentSystemName() ": A_RELEASE.REQ"
eventdown P_RELEASE.REQ
```

## A_RELEASE.RESP
```
;параметры: apcon (строка)
out CurrentSystemName() ": A_RELEASE.RESP apcon = " $apcon
 eventdown P_RELEASE.RESP
```

## A_RESOLVE.IND.DT
```
addrrr unbufferit lg_addr 1 lg_char1 1 lg_char2 1
out CurrentSystemName() ": A_RESOLVE.IND.DT lg_addr = " $lg_addr " lg_char1 = " $lg_char1 " lg_char2 = " $lg_char2

namer_buf pack $lg_addr 1 $_namer sizeof(cur_sys_name) sizeof(cur_sys_name)+1

timeevent A_ASSOCIATE.REQ func_timer 100 namer $namer_buf namei $namei_buf quality $_quality demand $gs_demand context $_context apcon "DT"
```

## A_RESOLVE.REQ
```
;параметры: name (строка)
out CurrentSystemName() ": A_RESOLVE.REQ name = " $name
$saved_addr != 0 if SAVED_ADDR

TRY_LOCGUIDE:
locguide($name) setto tmp_buf
tmp_buf unbufferit lg_type 1 lg_addr 1 lg_time1 1 lg_time2 1 lg_char1 1 lg_char2 1
out CurrentSystemName() ": GET_FROM_LG name = " $name " lg_type = " $lg_type " lg_addr = " $lg_addr " lg_time1 = " $lg_time1 " lg_time2 = " $lg_time2 " lg_char1 = " $lg_char1 " lg_char2 = " $lg_char2
$lg_type == 0 if got_addr
$lg_type == 1 if wait_time
$lg_type == 2 if wait_time
$lg_type == 3 if not_got_addr
$lg_type == 4 if not_got_addr
return

got_addr:
$lg_addr setto saved_addr
tmp_buf pack $lg_addr 1 0 1 " " 1 " " 1 4
timeevent A_RESOLVE.IND.DT func_timer 0 address $tmp_buf
return

wait_time:
timeevent A_RESOLVE.REQ func_timer $lg_time1 name $name
return

not_got_addr:
$name setto name_to_resolve
locguide("Guide") setto tmp_buf
tmp_buf unbufferit lg_type 1 lg_addr 1 lg_time1 1 lg_time2 1 lg_char1 1 lg_char2 1
out CurrentSystemName() ": GET_FROM_LG name = " "Guide" " lg_type = " $lg_type " lg_addr = " $lg_addr " lg_time1 = " $lg_time1 " lg_time2 = " $lg_time2 " lg_char1 = " $lg_char1 " lg_char2 = " $lg_char2
namer_buf pack $lg_addr 1 "Guide" 5 6

CurrentSystemName() setto cur_sys_name
locguide($cur_sys_name) setto tmp_buf
tmp_buf unbufferit lg_type 1 lg_addr 1 lg_time1 1 lg_time2 1 lg_char1 1 lg_char2 1
out CurrentSystemName() ": GET_FROM_LG name = " $cur_sys_name " lg_type = " $lg_type " lg_addr = " $lg_addr " lg_time1 = " $lg_time1 " lg_time2 = " $lg_time2 " lg_char1 = " $lg_char1 " lg_char2 = " $lg_char2
namei_buf pack $lg_addr 1 $cur_sys_name sizeof(cur_sys_name) 1+sizeof(cur_sys_name)

timeevent A_ASSOCIATE.REQ func_timer 0 namer $namer_buf namei $namei_buf quality $gs_quality demand $gs_demand context $gs_context apcon "GS"
return

SAVED_ADDR:
tmp_buf pack $saved_addr 1 0 1 " " 1 " " 1 4
timeevent A_RESOLVE.IND.DT func_timer 0 addrrr $tmp_buf
return


```

## A_SYNC_MAJOR.REQ
```
out CurrentSystemName() ": A_SYNC_MAJOR.REQ "
eventdown P_SYNC_MAJOR.REQ
```

## A_TERMINATE.RESP
```
out CurrentSystemName() ": A_TERMINATE.RESP "
timeevent A_RELEASE.RESP func_timer 0 apcon "DT"
```

## A_TERMINATE.REQ
```
;параметры: нет
out CurrentSystemName() ": A_TERMINATE.REQ "
timeevent A_RELEASE.REQ func_timer 0 apcon "DT"

```

## A_TRANSFER_INIT.REQ
```
;параметры: namer (строка), namei (строка), context (буфер), quality (буфер)
out CurrentSystemName() ": A_TRANSFER_INIT.REQ namer = " $namer " namei = " $namei  " context = " $context " quility = " $quality 
$namer setto _namer
$namei setto _namei
$context setto _context
$quality setto _quality
timeevent A_RESOLVE.REQ func_timer 0 name $namer


```

## A_TRANSFER_INIT.RESP
```
;параметры: namei (строка), context (буфер), quality (буфер)
out CurrentSystemName() ": A_TRANSFER_INIT.RESP namei = " $namei 
namei_buf pack $saved_addr 1 $_namei sizeof(_namei) sizeof(_namei)+1
timeevent A_ASSOCIATE.RESP func_timer 0  namei $namei_buf context $context quality $_quality apcon "DT"
```

## P_CONNECT.IND
```
;параметры:  address (число), quality (буфер), demand (буфер)
out CurrentSystemName() ": P_CONNECT.IND "

$address setto saved_addr
$quality setto _quality
$demand setto _demand
tmp_buf pack 3 1 $con_req_type 1 2

;address $address quality $quality context $gs_context eventdown P_CONNECT.RESP
userdata $tmp_buf eventdown P_EXPEDITED_DATA.REQ
;timeevent P_EXPEDITED_DATA.REQ.WAIT func_timer 50 userdata $tmp_buf

```

## P_CONNECT.CONF
```
;параметры:  quality (буфер), context (буфер)
out CurrentSystemName() ": P_CONNECT.CONF asos_apcon = " $asos_apcon
$asos_apcon == "GS" if GS_CONF
$asos_apcon == "DT" if DT_CONF
return

GS_CONF:
tmp_buf pack 3 1 $name_to_resolve sizeof(name_to_resolve) sizeof(name_to_resolve)+1
timeevent A_DATA.REQ func_timer 0 userdata $tmp_buf
return

DT_CONF:
sendup context $context quality $quality A_TRANSFER_INIT.CONF
return
```

## P_DATA.IND
```
;параметры:  userdata (буфер)
out CurrentSystemName() ": P_DATA.IND "
$asos_apcon == "GS" if GS_DATA
$asos_apcon == "DT" if DT_DATA
return
GS_DATA:
userdata unbufferit d_type 1 lg_type 1 lg_addr 1 lg_time1 1 lg_time2 1 lg_char1 1 lg_char2 1
out CurrentSystemName() ": GET_FROM_GUIDE name = " $name_to_resolve " lg_type = " $lg_type " lg_addr = " $lg_addr " lg_time1 = " $lg_time1 " lg_time2 = " $lg_time2 " lg_char1 = " $lg_char1 " lg_char2 = " $lg_char2
$lg_type == 0 if got_addr
$lg_type == 1 if wait_time
$lg_type == 2 if wait_time
$lg_type == 3 if not_got_addr
$lg_type == 4 if not_got_addr

got_addr:
$lg_addr setto saved_addr
got_addr_buf pack $lg_addr 1 0 1 " " 1 " " 1 4
eventdown P_RELEASE.REQ
return

wait_time:
tmp_buf pack 3 1 $name_to_resolve sizeof(name_to_resolve) sizeof(name_to_resolve)+1
timeevent A_DATA.REQ func_timer $lg_time1 userdata $tmp_buf
return

DT_DATA:
sendup userdata $userdata A_DATA.IND
return


```

## P_EXPEDITED_DATA.IND
```
userdata unbufferit pack_type 1 d_type 1 tmp_buf sizeof(userdata)-2
out CurrentSystemName() ": P_EXPEDITED_DATA.IND pack_type = " $pack_type " d_type = " $d_type
$d_type == $con_req_type if CON_REQ
$d_type == $con_res_type if CON_RES
return

CON_REQ:
send_buf pack 3 1 $con_res_type 1 $asos_apcon sizeof(asos_apcon) sizeof(asos_apcon) + 2
userdata $send_buf eventdown P_EXPEDITED_DATA.REQ
return

CON_RES:
tmp_buf unbufferit  asos_apcon sizeof(tmp_buf)
$asos_apcon== "DT" if gt_con
return

gt_con:
locguide($saved_addr) setto _namei
sendup namei $_namei quality $_quality A_TRANSFER_INIT.IND
return
 
```

## P_EXPEDITED_DATA.REQ.WAIT
```
out CurrentSystemName() ": P_EXPEDITED_DATA.REQ.WAIT "
userdata $userdata  eventdown P_EXPEDITED_DATA.REQ
```

## P_GIVE_TOKENS.IND
```
;параметры:  token (число)
out CurrentSystemName() ": P_GIVE_TOKENS.IND token = " $token
($token == 1) if resend_last_data
($token == 2) if try_sync
return
resend_last_data:
qcount(data_q) == 0 if break
userdata dequeue(data_q) eventdown P_DATA.REQ 
goto resend_last_data
return

try_sync:
eventdown P_SYNC_MAJOR.REQ
return

break:
token 2 eventdown P_PLEASE_TOKENS.REQ
return
```

## P_P_EXCEPTION.IND
```
;параметры:  error (число)
out CurrentSystemName() ": P_P_EXCEPTION.IND error = " $error
$error == 1 if need_data_t
$error == 2 if need_sync_t
return

need_data_t:
qcount(data_q) == 0 if break
userdata dequeue(data_q) eventdown P_DATA.REQ 
goto need_data_t
return

need_sync_t:
eventdown P_SYNC_MAJOR.REQ
return

break:
token 2 eventdown P_PLEASE_TOKENS.REQ
return
```

## P_PLEASE_TOKENS.IND
```
;параметры:  token (число)
out CurrentSystemName() ": P_PLEASE_TOKENS.IND "
token $token eventdown P_GIVE_TOKENS.REQ
$token !=  1 if skip
clearqueue data_q
skip:
return 
```

## P_RELEASE.CONF
```
;параметры:  нет
out CurrentSystemName() ": P_RELEASE.CONF asos_apcon = " $asos_apcon " saved_addr = " $saved_addr
$asos_apcon == "GS" if GS_RELEASE
$asos_apcon == "DT" if DT_RELEASE
return

GS_RELEASE:
timeevent A_RESOLVE.IND.DT func_timer 0 addrrr $got_addr_buf
return

DT_RELEASE:
sendup A_TERMINATE.CONF
```

## P_RELEASE.IND
```
;параметры:  нет
out CurrentSystemName() ": A_RELEASE.IND apcon = " $asos_apcon

$asos_apcon == "DT" if DT_RELEASE
return

DT_RELEASE:
sendup A_TERMINATE.IND
return
```

## P_SYNC_MAJOR.CONF
```
;параметры:  нет
out CurrentSystemName() ": P_SYNC_MAJOR.CONF "

```

## P_SYNC_MAJOR.IND
```
;параметры:  нет
out CurrentSystemName() ": P_SYNC_MAJOR.IND "
eventdown P_SYNC_MAJOR.RESP
```
