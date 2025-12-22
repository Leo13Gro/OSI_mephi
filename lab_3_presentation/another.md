# Уровень представления
## P_INIT.REQ
'''

buffer declare quality_buf
buffer declare demand_buf
buffer declare context_buf
buffer declare syntaxes 
buffer declare send_buf
buffer declare tmp_buf
buffer declare ttmp_buf
buffer declare empty_buf
buffer declare tmp_userdata

integer declare addr
integer declare s1
integer declare s2
integer declare syn
integer declare cur_syn
integer declare tmp_syn
integer declare data_type

string declare open_splitter 
string declare close_splitter
string declare word
string declare tmp_string

string declare splitter
"/" setto splitter


integer declare pack_type
integer declare connect_req_type
integer declare connect_res_type
integer declare data_req_type

1 setto connect_req_type
2 setto connect_res_type
3 setto data_req_type
'''

## P_CONNECT.REQ
'''
;параметры:  address (число), quality (буфер), demand (буфер), context (буфер)
out CurrentSystemName() ": P_CONNECT.REQ " 

;context unbufferit open_splitter 1 close_splitter 1 syntaxes sizeof(context) - 2
$context setto context_buf
$address setto addr
address $address  quality $quality demand $demand eventdown S_CONNECT.REQ
'''

## P_CONNECT.RESP
'''
;параметры:  address (число), quality (буфер), context (буфер)
out CurrentSystemName() ": P_CONNECT.RESP "

$address setto addr
$context setto context_buf
out CurrentSystemName() ": SIZEOF CONETEXT_BUF = " sizeof(context_buf)

context unbufferit open_splitter 1 close_splitter 1
out "mmm " $open_splitter ", " $close_splitter

address $address quality $quality  eventdown S_CONNECT.RESP
'''

## P_DATA.REQ
'''
;параметры:  userdata (буфер)
;out CurrentSystemName() ": P_DATA.REQ " 
userdata unbufferit data_type 1 tmp_buf sizeof(userdata)-1
out CurrentSystemName() ": P_DATA.REQ DATA_TYPE=" $data_type " SYNTAX=" $cur_syn
$data_type == 1 if STRUCT
userdata $userdata eventdown S_DATA.REQ
return

STRUCT:
context_buf unbufferit open_splitter 1 close_splitter 1 syntaxes sizeof(context_buf) - 2
tmp_buf unbufferit tmp_string sizeof(tmp_buf)
$cur_syn == 1 if len_type
split_type:
$empty_buf setto send_buf 
delete tmp_string 1 1

split_while:
sizeof(tmp_string ) == 1 if break
delete tmp_string 1 1
copy(tmp_string, 1, pos($close_splitter, tmp_string)-1) setto word
out CurrentSystemName() ": WORD=" $word ", " $close_splitter
delete tmp_string 1 sizeof(word)+1
send_buf pack $send_buf sizeof(send_buf) $word sizeof(word) $splitter sizeof(splitter) sizeof(send_buf)+sizeof(splitter)+sizeof(word)
goto split_while



return
len_type:
$empty_buf setto send_buf 
delete tmp_string 1 1

len_while:
sizeof(tmp_string ) == 1 if break
delete tmp_string 1 1
copy(tmp_string , 1, pos($close_splitter)) setto word
out CurrentSystemName() ": WORD=" $word
send_buf pack $send_buf sizeof(send_buff) sizeof(word) 1 word sizeof(word) sizeof(send_buff)+1+sizeof(word)
delete tmp_string 1 1
goto len_while
break:
send_buf pack $data_type 1 $cur_syn 1 $send_buf sizeof(send_buf) 2+sizeof(send_buf)
userdata $send_buf eventdown S_DATA.REQ

return
'''

## P_EXPEDITED_DATA.REQ
'''
;параметры:  userdata (буфер)
userdata unbufferit data_type 1 tmp_buf sizeof(userdata)-1
out CurrentSystemName() ": P_EXPEDITED_DATA.REQ DATA_TYPE=" $data_type " SYNTAX=" $cur_syn
$data_type == 1 if STRUCT
tmp_userdata pack $data_req_type 1 $userdata sizeof(userdata) 1+sizeof(userdata)
userdata $tmp_userdata eventdown S_EXPEDITED_DATA.REQ
return

STRUCT:
context_buf unbufferit open_splitter 1 close_splitter 1 syntaxes sizeof(context_buf) - 2
tmp_buf unbufferit tmp_string sizeof(tmp_buf)
$cur_syn == 1 if len_type
split_type:
$empty_buf setto send_buf 
delete tmp_string 1 1

split_while:
sizeof(tmp_string ) == 1 if break
delete tmp_string 1 1
copy(tmp_string, 1, pos($close_splitter, tmp_string)-1) setto word
out CurrentSystemName() ": WORD=" $word ", " $close_splitter
delete tmp_string 1 sizeof(word)+1
send_buf pack $send_buf sizeof(send_buf) $word sizeof(word) $splitter sizeof(splitter) sizeof(send_buf)+sizeof(splitter)+sizeof(word)
goto split_while



return
len_type:
$empty_buf setto send_buf 
delete tmp_string 1 1

len_while:
sizeof(tmp_string ) == 1 if break
delete tmp_string 1 1
copy(tmp_string , 1, pos($close_splitter)) setto word
out CurrentSystemName() ": WORD=" $word
send_buf pack $send_buf sizeof(send_buff) sizeof(word) 1 word sizeof(word) sizeof(send_buff)+1+sizeof(word)
delete tmp_string 1 1
goto len_while
break:
send_buf pack $data_req_type 1 $data_type 1 $cur_syn 1 $send_buf sizeof(send_buf) 3+sizeof(send_buf)
userdata $send_buf eventdown S_EXPEDITED_DATA.REQ

return
'''

## P_GIVE_TOKENS.REQ
'''
;параметры:  token (число)
out CurrentSystemName() ": P_GIVE_TOKENS.REQ " 
token $token eventdown S_GIVE_TOKENS.REQ
'''

## P_PLEASE_TOKENS.REQ
'''
;параметры:  token (число)
out CurrentSystemName() ": P_PLEASE_TOKENS.REQ " 
token $token eventdown S_PLEASE_TOKENS.REQ
'''

## P_RELEASE.REQ
'''
;параметры:  нет
out CurrentSystemName() ": P_RELEASE.REQ " 
eventdown S_RELEASE.REQ
'''

## P_RELEASE.RESP
'''
;параметры:  нет
out CurrentSystemName() ": P_RELEASE.RESP " 
eventdown S_RELEASE.RESP
'''

## P_SYNC_MAJOR.REQ
'''
;параметры:  нет 
out CurrentSystemName() ": P_SYNC_MAJOR.REQ " 
eventdown S_SYNC_MAJOR.REQ
'''

## P_SYNC_MAJOR.RESP
'''
;параметры:  нет
out CurrentSystemName() ": P_SYNC_MAJOR.RESP " 
eventdown S_SYNC_MAJOR.RESP
'''

## P_U_ABORT.REQ
'''
;параметры:  нет
out CurrentSystemName() ": P_U_ABORT.REQ "
eventdown S_U_ABORT.REQ
'''

## S_CONNECT.CONF
'''
;параметры:  quality (буфер)
out CurrentSystemName() ": S_CONNECT.CONF "

$quality setto quality_buf
out CurrentSystemName() ": SIZEOF CONTEXT_BUF = " sizeof(context_buf)

send_buf pack $connect_req_type 1 $context_buf sizeof(context_buf) 1+sizeof(context_buf)
userdata $send_buf  eventdown S_EXPEDITED_DATA.REQ
'''

## S_CONNECT.IND
'''
;параметры:  address (число), quality (буфер), demand (буфер)
out CurrentSystemName() ": S_CONNECT.IND " 

sendup address $address quality $quality demand $demand P_CONNECT.IND

'''

## S_DATA.IND
'''
;параметры:  userdata (буфер)
userdata unbufferit data_type 1 tmp_buf sizeof(userdata)-1
out CurrentSystemName() ": S_DATA.IND DATA_TYPE=" $data_type
$data_type == 1 if STRUCT
sendup userdata $userdata P_DATA.IND
return

STRUCT:
context_buf unbufferit open_splitter 1 close_splitter 1 syntaxes sizeof(context_buf) - 2
tmp_buf unbufferit tmp_syn 1 tmp_buf sizeof(tmp_buf)-1
tmp_buf unbufferit tmp_string sizeof(tmp_buf)
out CurrentSystemName() ": S_DATA.IND SYNTAX=" $tmp_syn " STRING = " $tmp_string
$tmp_syn == 1 if len_type
split_type:
$empty_buf setto send_buf 
send_buf pack $open_splitter sizeof(open_splitter) sizeof(open_splitter) 
split_while:
sizeof(tmp_string) == 0 if break
out CurrentSystemName() ": POS_WORD=" pos($splitter, tmp_string) " SPLITTER = " $splitter " STRING = " $tmp_string
copy(tmp_string, 1, pos($splitter, tmp_string)-1) setto word
out CurrentSystemName() ": WORD=" $word ", " $close_splitter
delete tmp_string 1 sizeof(word)+1
send_buf pack $send_buf sizeof(send_buf) $open_splitter sizeof(open_splitter) $word sizeof(word) $close_splitter sizeof(close_splitter) sizeof(send_buf)+sizeof(open_splitter)+sizeof(word)+sizeof(close_splitter)
goto split_while

return

len_type:
$empty_buf setto send_buf 
send_buf pack $open_splitter sizeof(open_splitter) sizeof(open_splitter) 
split_while:
sizeof(tmp_string) == 0 if break
copy(tmp_string, 1, pos($splitter, tmp_string)-1) setto word
out CurrentSystemName() ": WORD=" $word ", " $close_splitter
delete tmp_string 1 sizeof(word)+1
send_buf pack $send_buf sizeof(send_buf) $open_splitter sizeof(open_splitter) $word sizeof(word) $close_splitter sizeof(close_splitter) sizeof(send_buf)+sizeof(open_splitter)+sizeof(word)+sizeof(close_splitter)
goto split_while

return

break:
send_buf pack $data_type 1 $send_buf sizeof(send_buf) $close_splitter sizeof(close_splitter) 1+sizeof(send_buf) + sizeof(close_splitter)
sendup userdata $send_buf  P_DATA.IND

return
'''

## S_EXPEDITED_DATA.IND
'''
;параметры:  userdata (буфер)
out CurrentSystemName() ": S_EXPEDITED_DATA.IND "

out CurrentSystemName() ": SIZEOF USERDATA = " sizeof(userdata)
userdata unbufferit pack_type 1 tmp_buf sizeof(userdata)-1
$pack_type == $connect_req_type if CONNECT_REQ
$pack_type == $connect_res_type if CONNECT_RES
$pack_type == $data_req_type if DATA_REQ_TYPE
out CurrentSystemName() ":WARNING S_EXPEDITED_DATA.IND "
return
CONNECT_REQ:
tmp_buf unbufferit open_splitter 1 close_splitter 1 syntaxes sizeof(tmp_buf) - 2
sizeof(syntaxes) == 1 if req_unpack_one
syntaxes unbufferit s1 1 s2 1
out CurrentSystemName() ": Syn1 =  " $s1 " Sin2 = " $s2
$s1 setto cur_syn
goto req_send
req_unpack_one:
syntaxes unbufferit cur_syn 1
req_send:
send_buf  pack $connect_res_type 1 $context_buf sizeof(context_buf) 1+sizeof(context_buf)
userdata $send_buf  eventdown S_EXPEDITED_DATA.REQ
return

CONNECT_RES:
tmp_buf unbufferit open_splitter 1 close_splitter 1 syntaxes sizeof(tmp_buf) - 2
sizeof(syntaxes) == 1 if res_unpack_one
syntaxes unbufferit s1 1 s2 1
out CurrentSystemName() ": Syn1 =  " $s1 " Sin2 = " $s2
$s1 setto cur_syn
goto res_conf
res_unpack_one:
syntaxes unbufferit cur_syn 1
out CurrentSystemName() ": Syn1 =  " $cur_syn
res_conf:
tmp_buf pack $open_splitter 1 $close_splitter 1 $cur_syn 1 3
sendup quality $quality_buf context $tmp_buf P_CONNECT.CONF
;sendup address $addr quality $quality_buf demand $demand_buf S_CONNECT.IND
return

DATA_REQ_TYPE:
out CurrentSystemName() ": S_EXPEDITED_DATA.IND TMP_BUF=" $tmp_buf
$tmp_buf setto ttmp_buf
tmp_buf unbufferit data_type 1 tmp_buf sizeof(tmp_buf)-1
out CurrentSystemName() ": S_EXPEDITED_DATA.IND DATA_TYPE=" $data_type
$data_type == 1 if STRUCT
sendup userdata $ttmp_buf  P_EXPEDITED_DATA.IND
return

STRUCT:
context_buf unbufferit open_splitter 1 close_splitter 1 syntaxes sizeof(context_buf) - 2
tmp_buf unbufferit tmp_syn 1 tmp_buf sizeof(tmp_buf)-1
tmp_buf unbufferit tmp_string sizeof(tmp_buf)
out CurrentSystemName() ": S_DATA.IND SYNTAX=" $tmp_syn " STRING = " $tmp_string
$tmp_syn == 1 if len_type
split_type:
$empty_buf setto send_buf 
send_buf pack $open_splitter sizeof(open_splitter) sizeof(open_splitter) 
split_while:
sizeof(tmp_string) == 0 if break
out CurrentSystemName() ": POS_WORD=" pos($splitter, tmp_string) " SPLITTER = " $splitter " STRING = " $tmp_string
copy(tmp_string, 1, pos($splitter, tmp_string)-1) setto word
out CurrentSystemName() ": WORD=" $word ", " $close_splitter
delete tmp_string 1 sizeof(word)+1
send_buf pack $send_buf sizeof(send_buf) $open_splitter sizeof(open_splitter) $word sizeof(word) $close_splitter sizeof(close_splitter) sizeof(send_buf)+sizeof(open_splitter)+sizeof(word)+sizeof(close_splitter)
goto split_while

return

len_type:
out "!!! len_type"
$empty_buf setto send_buf 
send_buf pack $open_splitter sizeof(open_splitter) sizeof(open_splitter) 
split_while:
sizeof(tmp_string) == 0 if break
copy(tmp_string, 1, pos($splitter, tmp_string)-1) setto word
out CurrentSystemName() ": WORD=" $word ", " $close_splitter
delete tmp_string 1 sizeof(word)+1
send_buf pack $send_buf sizeof(send_buf) $open_splitter sizeof(open_splitter) $word sizeof(word) $close_splitter sizeof(close_splitter) sizeof(send_buf)+sizeof(open_splitter)+sizeof(word)+sizeof(close_splitter)
goto split_while

return

break:
send_buf pack $data_type 1 $send_buf sizeof(send_buf) $close_splitter sizeof(close_splitter) 1+sizeof(send_buf) + sizeof(close_splitter)
sendup userdata $send_buf  P_DATA.IND

return


return
'''

## S_GIVE_TOKENS.IND
'''
;параметры:  token (число)
out CurrentSystemName() ": S_GIVE_TOKENS.IND " 
sendup token $token P_GIVE_TOKENS.IND
'''

## S_P_EXCEPTION.IND
'''
;параметры:  error (число)
out CurrentSystemName() ": S_P_EXCEPTION.IND " 
sendup error $error P_P_EXCEPTION.IND
'''

## S_PLEASE_TOKENS.IND
'''
;параметры:  token (число)
out CurrentSystemName() ": S_PLEASE_TOKENS.IND " 
sendup token $token P_PLEASE_TOKENS.IND
'''

## S_RELEASE.CONF
'''
;параметры:  нет
out CurrentSystemName() ": S_RELEASE.CONF " 
sendup P_RELEASE.CONF
'''

## S_RELEASE.IND
'''
;параметры:  нет
out CurrentSystemName() ": S_RELEASE.IND " 
sendup P_RELEASE.IND
'''

## S_SYNC_MAJOR.CONF
'''
;параметры:  нет
out CurrentSystemName() ": S_SYNC_MAJOR.CONF " 
sendup P_SYNC_MAJOR.CONF
'''

## S_SYNC_MAJOR.IND
'''
;параметры:  нет
out CurrentSystemName() ": S_SYNC_MAJOR.IND " 
sendup P_SYNC_MAJOR.IND
'''

## S_U_ABORT.IND
'''
;параметры:  нет
out CurrentSystemName() ": S_U_ABORT.IND "
sendup P_U_ABORT.IND
'''