# Уровень представления
## P_INIT.REQ
```
quality_buf declare buffer
demand_buf declare buffer
context_buf declare buffer
syntaxes declare buffer
send_buf declare buffer
tmp_buf declare buffer
ttmp_buf declare buffer
empty_buf declare buffer
tmp_userdata declare buffer

addr declare integer
s1 declare integer
s2 declare integer
syn declare integer
cur_syn declare integer
tmp_syn declare integer
data_type declare integer

open_splitter declare string
close_splitter declare string
word declare string
tmp_string declare string

splitter declare string
"/" varset splitter

pack_type declare integer
connect_req_type declare integer
connect_res_type declare integer
data_req_type declare integer

1 varset connect_req_type
2 varset connect_res_type
3 varset data_req_type
```

## P_CONNECT.REQ
```
;параметры:  address (число), quality (буфер), demand (буфер), context (буфер)
out CurrentSystemName() + ": P_CONNECT.REQ "

$context varset context_buf
$address varset addr

S_CONNECT.REQ eventdown { address $address quality $quality demand $demand }
```

## P_CONNECT.RESP
```
;параметры:  address (число), quality (буфер), context (буфер)
out CurrentSystemName() + ": P_CONNECT.RESP "

$address varset addr
$context varset context_buf
out CurrentSystemName() + ": SIZEOF CONETEXT_BUF = " + sizeof(context_buf)

S_CONNECT.RESP eventdown { address $address quality $quality }
```

## P_DATA.REQ
```
;параметры:  userdata (буфер)
unbufferit userdata { data_type 1 tmp_buf sizeof(userdata)-1 }
out CurrentSystemName() + ": P_DATA.REQ DATA_TYPE=" + $data_type + " SYNTAX=" + $cur_syn

$data_type == 1 if STRUCT
S_DATA.REQ eventdown { userdata $userdata }
return

STRUCT:
unbufferit context_buf { open_splitter 1 close_splitter 1 syntaxes sizeof(context_buf) - 2 }
unbufferit tmp_buf { tmp_string sizeof(tmp_buf) }

$cur_syn == 1 if len_type

split_type:
$empty_buf varset send_buf
delete tmp_string 1 1

split_while:
sizeof(tmp_string) == 1 if break
delete tmp_string 1 1
copy(tmp_string, 1, pos($close_splitter, tmp_string)-1) varset word
out CurrentSystemName() + ": WORD=" + $word + ", " + $close_splitter
delete tmp_string 1 sizeof(word)+1

send_buf bufferit sizeof(send_buf)+sizeof(splitter)+sizeof(word) {
  $send_buf sizeof(send_buf)
  $word sizeof(word)
  $splitter sizeof(splitter)
}

goto split_while

return

len_type:
$empty_buf varset send_buf
delete tmp_string 1 1

len_while:
sizeof(tmp_string) == 1 if break
delete tmp_string 1 1
copy(tmp_string, 1, pos($close_splitter, tmp_string)) varset word
out CurrentSystemName() + ": WORD=" + $word

send_buf bufferit sizeof(send_buf)+1+sizeof(word) {
  $send_buf sizeof(send_buf)
  sizeof(word) 1
  $word sizeof(word)
}

delete tmp_string 1 1
goto len_while

break:
send_buf bufferit 2+sizeof(send_buf) {
  $data_type 1
  $cur_syn 1
  $send_buf sizeof(send_buf)
}

S_DATA.REQ eventdown { userdata $send_buf }
return
```

## P_EXPEDITED_DATA.REQ
```
;параметры:  userdata (буфер)
unbufferit userdata { data_type 1 tmp_buf sizeof(userdata)-1 }
out CurrentSystemName() + ": P_EXPEDITED_DATA.REQ DATA_TYPE=" + $data_type + " SYNTAX=" + $cur_syn

$data_type == 1 if STRUCT

tmp_userdata bufferit 1+sizeof(userdata) { $data_req_type 1 $userdata sizeof(userdata) }
S_EXPEDITED_DATA.REQ eventdown { userdata $tmp_userdata }
return

STRUCT:
unbufferit context_buf { open_splitter 1 close_splitter 1 syntaxes sizeof(context_buf) - 2 }
unbufferit tmp_buf { tmp_string sizeof(tmp_buf) }

$cur_syn == 1 if len_type

split_type:
$empty_buf varset send_buf
delete tmp_string 1 1

split_while:
sizeof(tmp_string) == 1 if break
delete tmp_string 1 1
copy(tmp_string, 1, pos($close_splitter, tmp_string)-1) varset word
out CurrentSystemName() + ": WORD=" + $word + ", " + $close_splitter
delete tmp_string 1 sizeof(word)+1

send_buf bufferit sizeof(send_buf)+sizeof(splitter)+sizeof(word) {
  $send_buf sizeof(send_buf)
  $word sizeof(word)
  $splitter sizeof(splitter)
}

goto split_while

return

len_type:
$empty_buf varset send_buf
delete tmp_string 1 1

len_while:
sizeof(tmp_string) == 1 if break
delete tmp_string 1 1
copy(tmp_string, 1, pos($close_splitter, tmp_string)) varset word
out CurrentSystemName() + ": WORD=" + $word

send_buf bufferit sizeof(send_buf)+1+sizeof(word) {
  $send_buf sizeof(send_buf)
  sizeof(word) 1
  $word sizeof(word)
}

delete tmp_string 1 1
goto len_while

break:
send_buf bufferit 3+sizeof(send_buf) {
  $data_req_type 1
  $data_type 1
  $cur_syn 1
  $send_buf sizeof(send_buf)
}

S_EXPEDITED_DATA.REQ eventdown { userdata $send_buf }
return
```

## P_GIVE_TOKENS.REQ
```
;параметры:  token (число)
out CurrentSystemName() + ": P_GIVE_TOKENS.REQ "
S_GIVE_TOKENS.REQ eventdown { token $token }
```

## P_PLEASE_TOKENS.REQ
```
;параметры:  token (число)
out CurrentSystemName() + ": P_PLEASE_TOKENS.REQ "
S_PLEASE_TOKENS.REQ eventdown { token $token }
```

## P_RELEASE.REQ
```
;параметры:  нет
out CurrentSystemName() + ": P_RELEASE.REQ "
S_RELEASE.REQ eventdown { }
```

## P_RELEASE.RESP
```
;параметры:  нет
out CurrentSystemName() + ": P_RELEASE.RESP "
S_RELEASE.RESP eventdown { }
```

## P_SYNC_MAJOR.REQ
```
;параметры:  нет
out CurrentSystemName() + ": P_SYNC_MAJOR.REQ "
S_SYNC_MAJOR.REQ eventdown { }
```

## P_SYNC_MAJOR.RESP
```
;параметры:  нет
out CurrentSystemName() + ": P_SYNC_MAJOR.RESP "
S_SYNC_MAJOR.RESP eventdown { }
```

## P_U_ABORT.REQ
```
;параметры:  нет
out CurrentSystemName() + ": P_U_ABORT.REQ "
S_U_ABORT.REQ eventdown { }
```

## S_CONNECT.CONF
```
;параметры:  quality (буфер)
out CurrentSystemName() + ": S_CONNECT.CONF "

$quality varset quality_buf
out CurrentSystemName() + ": SIZEOF CONTEXT_BUF = " + sizeof(context_buf)

send_buf bufferit 1+sizeof(context_buf) { $connect_req_type 1 $context_buf sizeof(context_buf) }
S_EXPEDITED_DATA.REQ eventdown { userdata $send_buf }
```

## S_CONNECT.IND
```
;параметры:  address (число), quality (буфер), demand (буфер)
out CurrentSystemName() + ": S_CONNECT.IND "
P_CONNECT.IND generateup { address $address quality $quality demand $demand }
```

## S_DATA.IND
```
;параметры:  userdata (буфер)
unbufferit userdata { data_type 1 tmp_buf sizeof(userdata)-1 }
out CurrentSystemName() + ": S_DATA.IND DATA_TYPE=" + $data_type

$data_type == 1 if STRUCT
P_DATA.IND generateup { userdata $userdata }
return

STRUCT:
unbufferit context_buf { open_splitter 1 close_splitter 1 syntaxes sizeof(context_buf) - 2 }

unbufferit tmp_buf { tmp_syn 1 tmp_buf sizeof(tmp_buf)-1 }
unbufferit tmp_buf { tmp_string sizeof(tmp_buf) }

out CurrentSystemName() + ": S_DATA.IND SYNTAX=" + $tmp_syn + " STRING = " + $tmp_string

$tmp_syn == 1 if len_type

split_type:
$empty_buf varset send_buf

send_buf bufferit sizeof(open_splitter) { $open_splitter sizeof(open_splitter) }

split_while:
sizeof(tmp_string) == 0 if break
out CurrentSystemName() + ": POS_WORD=" + pos($splitter, tmp_string) + " SPLITTER = " + $splitter + " STRING = " + $tmp_string

copy(tmp_string, 1, pos($splitter, tmp_string)-1) varset word
out CurrentSystemName() + ": WORD=" + $word + ", " + $close_splitter
delete tmp_string 1 sizeof(word)+1

send_buf bufferit sizeof(send_buf)+sizeof(open_splitter)+sizeof(word)+sizeof(close_splitter) {
  $send_buf sizeof(send_buf)
  $open_splitter sizeof(open_splitter)
  $word sizeof(word)
  $close_splitter sizeof(close_splitter)
}

goto split_while

return

len_type:
$empty_buf varset send_buf

send_buf bufferit sizeof(open_splitter) { $open_splitter sizeof(open_splitter) }

split_while:
sizeof(tmp_string) == 0 if break

copy(tmp_string, 1, pos($splitter, tmp_string)-1) varset word
out CurrentSystemName() + ": WORD=" + $word + ", " + $close_splitter
delete tmp_string 1 sizeof(word)+1

send_buf bufferit sizeof(send_buf)+sizeof(open_splitter)+sizeof(word)+sizeof(close_splitter) {
  $send_buf sizeof(send_buf)
  $open_splitter sizeof(open_splitter)
  $word sizeof(word)
  $close_splitter sizeof(close_splitter)
}

goto split_while

return

break:
send_buf bufferit 1+sizeof(send_buf)+sizeof(close_splitter) {
  $data_type 1
  $send_buf sizeof(send_buf)
  $close_splitter sizeof(close_splitter)
}

P_DATA.IND generateup { userdata $send_buf }
return
```

## S_EXPEDITED_DATA.IND
```
;параметры:  userdata (буфер)
out CurrentSystemName() + ": S_EXPEDITED_DATA.IND "
out CurrentSystemName() + ": SIZEOF USERDATA = " + sizeof(userdata)

unbufferit userdata { pack_type 1 tmp_buf sizeof(userdata)-1 }

$pack_type == $connect_req_type if CONNECT_REQ
$pack_type == $connect_res_type if CONNECT_RES
$pack_type == $data_req_type if DATA_REQ_TYPE

out CurrentSystemName() + ":WARNING S_EXPEDITED_DATA.IND "
return

CONNECT_REQ:
unbufferit tmp_buf { open_splitter 1 close_splitter 1 syntaxes sizeof(tmp_buf) - 2 }

sizeof(syntaxes) == 1 if req_unpack_one
unbufferit syntaxes { s1 1 s2 1 }
out CurrentSystemName() + ": Syn1 =  " + $s1 + " Sin2 = " + $s2
$s1 varset cur_syn
goto req_send

req_unpack_one:
unbufferit syntaxes { cur_syn 1 }

req_send:
send_buf bufferit 1+sizeof(context_buf) { $connect_res_type 1 $context_buf sizeof(context_buf) }
S_EXPEDITED_DATA.REQ eventdown { userdata $send_buf }
return

CONNECT_RES:
unbufferit tmp_buf { open_splitter 1 close_splitter 1 syntaxes sizeof(tmp_buf) - 2 }

sizeof(syntaxes) == 1 if res_unpack_one
unbufferit syntaxes { s1 1 s2 1 }
out CurrentSystemName() + ": Syn1 =  " + $s1 + " Sin2 = " + $s2
$s1 varset cur_syn
goto res_conf

res_unpack_one:
unbufferit syntaxes { cur_syn 1 }
out CurrentSystemName() + ": Syn1 =  " + $cur_syn

res_conf:
tmp_buf bufferit 3 { $open_splitter 1 $close_splitter 1 $cur_syn 1 }
P_CONNECT.CONF generateup { quality $quality_buf context $tmp_buf }
return

DATA_REQ_TYPE:
out CurrentSystemName() + ": S_EXPEDITED_DATA.IND TMP_BUF=" + $tmp_buf
$tmp_buf varset ttmp_buf

unbufferit tmp_buf { data_type 1 tmp_buf sizeof(tmp_buf)-1 }
out CurrentSystemName() + ": S_EXPEDITED_DATA.IND DATA_TYPE=" + $data_type

$data_type == 1 if STRUCT
P_EXPEDITED_DATA.IND generateup { userdata $ttmp_buf }
return

STRUCT:
unbufferit context_buf { open_splitter 1 close_splitter 1 syntaxes sizeof(context_buf) - 2 }

unbufferit tmp_buf { tmp_syn 1 tmp_buf sizeof(tmp_buf)-1 }
unbufferit tmp_buf { tmp_string sizeof(tmp_buf) }

out CurrentSystemName() + ": S_DATA.IND SYNTAX=" + $tmp_syn + " STRING = " + $tmp_string

$tmp_syn == 1 if len_type

split_type:
$empty_buf varset send_buf
send_buf bufferit sizeof(open_splitter) { $open_splitter sizeof(open_splitter) }

split_while:
sizeof(tmp_string) == 0 if break
out CurrentSystemName() + ": POS_WORD=" + pos($splitter, tmp_string) + " SPLITTER = " + $splitter + " STRING = " + $tmp_string

copy(tmp_string, 1, pos($splitter, tmp_string)-1) varset word
out CurrentSystemName() + ": WORD=" + $word + ", " + $close_splitter
delete tmp_string 1 sizeof(word)+1

send_buf bufferit sizeof(send_buf)+sizeof(open_splitter)+sizeof(word)+sizeof(close_splitter) {
  $send_buf sizeof(send_buf)
  $open_splitter sizeof(open_splitter)
  $word sizeof(word)
  $close_splitter sizeof(close_splitter)
}

goto split_while
return

len_type:
$empty_buf varset send_buf
send_buf bufferit sizeof(open_splitter) { $open_splitter sizeof(open_splitter) }

split_while:
sizeof(tmp_string) == 0 if break

copy(tmp_string, 1, pos($splitter, tmp_string)-1) varset word
out CurrentSystemName() + ": WORD=" + $word + ", " + $close_splitter
delete tmp_string 1 sizeof(word)+1

send_buf bufferit sizeof(send_buf)+sizeof(open_splitter)+sizeof(word)+sizeof(close_splitter) {
  $send_buf sizeof(send_buf)
  $open_splitter sizeof(open_splitter)
  $word sizeof(word)
  $close_splitter sizeof(close_splitter)
}

goto split_while
return

break:
send_buf bufferit 1+sizeof(send_buf)+sizeof(close_splitter) {
  $data_type 1
  $send_buf sizeof(send_buf)
  $close_splitter sizeof(close_splitter)
}

P_DATA.IND generateup { userdata $send_buf }
return
```

## S_GIVE_TOKENS.IND
```
;параметры:  token (число)
out CurrentSystemName() + ": S_GIVE_TOKENS.IND "
P_GIVE_TOKENS.IND generateup { token $token }
```

## S_P_EXCEPTION.IND
```
;параметры:  error (число)
out CurrentSystemName() + ": S_P_EXCEPTION.IND "
P_P_EXCEPTION.IND generateup { error $error }
```

## S_PLEASE_TOKENS.IND
```
;параметры:  token (число)
out CurrentSystemName() + ": S_PLEASE_TOKENS.IND "
P_PLEASE_TOKENS.IND generateup { token $token }
```

## S_RELEASE.CONF
```
;параметры:  нет
out CurrentSystemName() + ": S_RELEASE.CONF "
P_RELEASE.CONF generateup { }
```

## S_RELEASE.IND
```
;параметры:  нет
out CurrentSystemName() + ": S_RELEASE.IND "
P_RELEASE.IND generateup { }
```

## S_SYNC_MAJOR.CONF
```
;параметры:  нет
out CurrentSystemName() + ": S_SYNC_MAJOR.CONF "
P_SYNC_MAJOR.CONF generateup { }
```

## S_SYNC_MAJOR.IND
```
;параметры:  нет
out CurrentSystemName() + ": S_SYNC_MAJOR.IND "
P_SYNC_MAJOR.IND generateup { }
```

## S_U_ABORT.IND
```
;параметры:  нет
out CurrentSystemName() + ": S_U_ABORT.IND "
P_U_ABORT.IND generateup { }
```
