

marble  => 250–300 ms.


 Get-Content .env | ForEach-Object { if ($_ -match '^\s*#' -or $_ -match '^\s*$') { return }; $kv = $_ -split '=', 2; if ($kv.Count -ge 2) { Set-Item -Path ("Env:" + $kv[0].Trim()) -Value $kv[1].Trim().Trim('"') } }



```sh
$env:DockerComposePostgresPassword = "SuperSecretPasswordToChangeForPg"
$env:DockerComposeRabbitMQPassword = "SuperSecretPasswordToChangeForAmqp"
$env:DockerComposeJWTKey = "IMPORTANT:_ChangeThisKey_~%pvif3KRo!3Mkm1oMC50TvAPi%{mUt<9sBm>DPjGZyfYYWssseVrNUqLQE}mz{L_UsingThisKeyIsDangerous"
$env:DockerComposePasswordHashingKey = "IMPORTANT:_ChangeThisKey_~%pvif3KRo!3Mkm1oMC50TvAPi%{mUt<9sBm>DPjGZyfYYWssseVrNUqLQE}mz{L_UsingThisKeyIsDangerous"

```