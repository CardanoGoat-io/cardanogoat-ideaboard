; Shelley Types

block =
  [ header
  , transaction_bodies         : [* transaction_body]
  , transaction_witness_sets   : [* transaction_witness_set]
  , transaction_metadata_set   : { * uint => transaction_metadata }
  ]

header =
  ( header_body
  , body_signature : $kes_signature
  )

header_body =
  ( prev_hash        : $hash
  , issuer_vkey      : $vkey
  , vrf_vkey         : $vrf_vkey
  , slot             : uint
  , nonce            : uint
  , nonce_proof      : $vrf_proof
  , leader_value     : unit_interval
  , leader_proof     : $vrf_proof
  , size             : uint
  , block_number     : uint
  , block_body_hash  : $hash            ; merkle pair root
  , operational_cert
  , protocol_version
  )

operational_cert =
  ( hot_vkey        : $kes_vkey
  , cold_vkey       : $vkey
  , sequence_number : uint
  , kes_period      : uint
  , sigma           : $signature
  )

protocol_version = (uint, uint, uint)

; Do we want to use a Map here? Is it actually cheaper?
; Do we want to add extension points here?
transaction_body =
  { 0 : #6.258([* transaction_input])
  , 1 : [* transaction_output]
  , ? 2 : [* delegation_certificate]
  , ? 3 : withdrawals
  , 4 : coin ; fee
  , 5 : uint ; ttl
  , ? 6 : full_update
  , ? 7 : metadata_hash
  }

; Is it okay to have this as a group? Is it valid CBOR?! Does it need to be?
transaction_input = [transaction_id : $hash, index : uint]

transaction_output = [address, amount : uint]

address =
 (  0, keyhash, keyhash       ; base address
 // 1, keyhash, scripthash    ; base address
 // 2, scripthash, keyhash    ; base address
 // 3, scripthash, scripthash ; base address
 // 4, keyhash, pointer       ; pointer address
 // 5, scripthash, pointer    ; pointer address
 // 6, keyhash                ; enterprise address (null staking reference)
 // 7, scripthash             ; enterprise address (null staking reference)
 // 8, keyhash                ; bootstrap address
 )

delegation_certificate =
  [( 0, keyhash                       ; stake key registration
  // 1, scripthash                    ; stake script registration
  // 2, keyhash                       ; stake key de-registration
  // 3, scripthash                    ; stake script de-registration
  // 4                                ; stake key delegation
      , keyhash                       ; delegating key
      , keyhash                       ; key delegated to
  // 5                                ; stake script delegation
      , scripthash                    ; delegating script
      , keyhash                       ; key delegated to
  // 6, keyhash, pool_params          ; stake pool registration
  // 7, keyhash, epoch                ; stake pool retirement
  // 8                                ; genesis key delegation
      , genesishash                   ; delegating key
      , keyhash                       ; key delegated to
  // 9, move_instantaneous_reward ; move instantaneous rewards
 ) ]

move_instantaneous_reward = { * keyhash => coin }
pointer = (uint, uint, uint)

credential =
  (  0, keyhash
  // 1, scripthash
  // 2, genesishash
  )

pool_params = ( #6.258([* keyhash]) ; pool owners
              , coin                ; cost
              , unit_interval       ; margin
              , coin                ; pledge
              , keyhash             ; operator
              , $vrf_keyhash        ; vrf keyhash
              , [credential]        ; reward account
              )

withdrawals = { * [credential] => coin }

full_update = [ protocol_param_update_votes, application_version_update_votes ]

protocol_param_update_votes =
  { * genesishash => protocol_param_update }

protocol_param_update =
  { ? 0:  uint               ; minfee A
  , ? 1:  uint               ; minfee B
  , ? 2:  uint               ; max block body size
  , ? 3:  uint               ; max transaction size
  , ? 4:  uint               ; max block header size
  , ? 5:  coin               ; key deposit
  , ? 6:  unit_interval      ; key deposit min refund
  , ? 7:  rational           ; key deposit decay rate
  , ? 8:  coin               ; pool deposit
  , ? 9:  unit_interval      ; pool deposit min refund
  , ? 10: rational           ; pool deposit decay rate
  , ? 11: epoch              ; maximum epoch
  , ? 12: uint               ; n_optimal. desired number of stake pools
  , ? 13: rational           ; pool pledge influence
  , ? 14: unit_interval      ; expansion rate
  , ? 15: unit_interval      ; treasury growth rate
  , ? 16: unit_interval      ; active slot coefficient
  , ? 17: unit_interval      ; d. decentralization constant
  , ? 18: uint               ; extra entropy
  , ? 19: [protocol_version] ; protocol version
  }

application_version_update_votes = { * genesishash => application_version_update }

application_version_update = { * application_name =>  [uint, application_metadata] }

application_metadata = { * system_tag => installerhash }

application_name = tstr .size 12
system_tag = tstr .size 10

transaction_witness_set =
  (  0, vkeywitness
  // 1, $script
  // 2, [* vkeywitness]
  // 3, [* $script]
  // 4, [* vkeywitness],[* $script]
  )

transaction_metadata =
    { * transaction_metadata => transaction_metadata }
  / [ * transaction_metadata ]
  / int
  / bytes
  / text

vkeywitness = [$vkey, $signature]

unit_interval = rational

rational =  #6.30(
   [ numerator   : uint
   , denominator : uint
   ])

coin = uint
epoch = uint

keyhash = $hash

scripthash = $hash

genesishash = $hash

installerhash = $hash

metadata_hash = $hash

$hash /= bytes

$vkey /= bytes

$signature /= bytes

$vrf_keyhash /= bytes

$vrf_vkey /= bytes
$vrf_proof /= bytes

$kes_vkey /= bytes

$kes_signature /= bytes