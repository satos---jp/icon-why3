とりあえずOvenから移植試してみる
- predicate中はaddrのbalanceってどう取得すればいいんだ
- utility 関数をcontract中に書くと Unexpected decl エラーになる...

- 
sp.transfer(argument: t, amount: sp.mutez, destination: sp.contract[t])
    Calls the contract at destination with argument while transferring amount to it.

- Xfer (i, m, a) :: ~o
Typically, i is used for a general integer, m for the amount of
XTZ (in a unit mutez), and a for the address of an account.


Gp’0default’0int 10 represents a tagged value passed
to a contract to invoke the default entrypoint with param-
eter 10.
