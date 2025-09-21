# Relatório: Mini-Projeto 1 - Quebra-Senhas Paralelo

**Aluno(s):** Caio Cesar Navarro Pugliese (10436413), Erik Dong Kyu Kang (10439715), Matheus Lemos Rosal do valle (10442011), Rodrigo Daiske Uehara (10440295).
---

## 1. Estratégia de Paralelização


**Como você dividiu o espaço de busca entre os workers?**

A paralelização foi feita dividindo o espaço total de senhas possíveis em faixas consecutivas de combinações, calculadas a partir de `total_space = charset_len^password_len`. Cada worker recebe uma parte desse espaço usando passwords_per_worker = total_space / num_workers, e o restante da divisão é distribuído entre os primeiros workers para manter o balanceamento. Os índices inicial e final de cada faixa são convertidos em senhas reais pela função index_to_password e passados como parâmetros ao worker via fork() e execl(). Assim, cada worker percorre apenas sua parte, sem sobreposição e com divisão equilibrada do trabalho.]

**Código relevante:** Cole aqui a parte do coordinator.c onde você calcula a divisão:
```c
long long passwords_per_worker = total_space / num_workers;
long long remaining = total_space % num_workers;

for (int i = 0; i < num_workers; i++) {
    // Cálculo do índice inicial do intervalo
    long long start_index = i * passwords_per_worker;
    if (i < remaining) {
        start_index += i;
    } else {
        start_index += remaining;
    }

    // Cálculo do índice final do intervalo
    long long end_index = start_index + passwords_per_worker - 1;
    if (i < remaining) {
        end_index++;
    }

    // Conversão dos índices em senhas reais
    char start_password[32];
    char end_password[32];
    index_to_password(start_index, charset, charset_len, password_len, start_password);
    index_to_password(end_index, charset, charset_len, password_len, end_password);

    // Criação do processo worker
    pid_t pid = fork();
    if (pid == 0) {
        execl("./worker", "./worker", target_hash, start_password, end_password,
              charset, password_len_str, worker_id_str, (char *)NULL);
        perror("Erro ao executar worker");
        exit(1);
    } else if (pid > 0) {
        workers[i] = pid;  // Pai guarda o PID do filho
    } else {
        perror("Erro ao criar processo");
        exit(1);
    }
}

```

---

## 2. Implementação das System Calls

**Descreva como você usou fork(), execl() e wait() no coordinator:**

No coordinator, utilizei fork() para criar os processos filhos responsáveis por executar os workers. Em cada iteração do loop, o processo filho chama execl() para substituir sua imagem de execução pelo programa do worker, recebendo como argumentos o hash alvo, o intervalo de senhas (start_password e end_password), o charset, o tamanho da senha e o ID do worker. Enquanto isso, o processo pai armazena o PID de cada filho e continua a criar os demais workers. Após todos serem criados, o coordinator usa wait() em um loop para aguardar a finalização de cada worker, verificando o status de saída para garantir a execução correta e evitar processos zumbis.

**Código do fork/exec:**
```c
for (int i = 0; i < num_workers; i++) {
    long long start_index = i * passwords_per_worker;
    if(i < remaining){
        start_index += i;
    } else {
        start_index += remaining;
    }
    long long end_index = start_index + passwords_per_worker - 1;
    if(i < remaining){
        end_index++;
    } 

    char start_password[32];
    char end_password[32];
    index_to_password(start_index, charset, charset_len, password_len, start_password);
    index_to_password(end_index, charset, charset_len, password_len, end_password);
    
    char password_len_str[16];
    char worker_id_str[16];
    sprintf(password_len_str, "%d", password_len);
    sprintf(worker_id_str, "%d", i);

    pid_t pid = fork();
    if(pid < 0){
        perror("Erro ao criar processo");
        exit(1);
    } else if(pid == 0){
        execl("./worker","./worker", target_hash, start_password, end_password,
              charset, password_len_str, worker_id_str, (char *)NULL);
        perror("Erro ao executar o worker");
        exit(1);
    } else{
        workers[i] = pid;
    }        
}

int workersTerminaram = 0;
while(workersTerminaram < num_workers){
    int status;
    pid_t wpid = wait(&status);
    if(wpid == -1){
        perror("Erro no wait");
        break;
    }
    int workerIndex = -1;
    for(int i = 0; i < num_workers; i++){
        if(workers[i] == wpid){
            workerIndex = i;
            break;
        }
    }
    
    if((WIFEXITED(status)) == 1){
        int codigoSaida = WEXITSTATUS(status);
        printf("O worker %d de PID %d terminou normalmente com o codigo %d\n", workerIndex, wpid, codigoSaida);
    } else{
        printf("O worker %d de PID %d terminou com erro", workerIndex, wpid);
    }
    workersTerminaram++;
}

```

---

## 3. Comunicação Entre Processos

**Como você garantiu que apenas um worker escrevesse o resultado?**

Para garantir que apenas um worker escrevesse o resultado, cada worker tenta criar o arquivo password_found.txt usando a flag O_CREAT | O_EXCL do open(). Essa combinação faz com que a criação do arquivo seja atômica: se o arquivo já existe, a chamada falha e o worker sabe que outro já salvou a senha. Dessa forma, evitam-se condições de corrida, pois nenhum worker sobrescreve o resultado de outro.

**Como o coordinator consegue ler o resultado?**

O coordinator consegue ler o resultado abrindo o arquivo password_found.txt para leitura. Ele lê o conteúdo, que está no formato worker_id:senha, e faz o parse separando o ID do worker e a senha encontrada. Em seguida, pode verificar o hash da senha lida usando md5_string() para confirmar que é o resultado correto e exibi-lo ao usuário.

---

## 4. Análise de Performance
Complete a tabela com tempos reais de execução:
O speedup é o tempo do teste com 1 worker dividido pelo tempo com 4 workers.

| Teste | 1 Worker | 2 Workers | 4 Workers | Speedup (4w) |
|-------|----------|-----------|-----------|--------------|
| Hash: 202cb962ac59075b964b07152d234b70<br>Charset: "0123456789"<br>Tamanho: 3<br>Senha: "123" | 0.004s | 0.005s | 0.004s | 1 |
| Hash: 5d41402abc4b2a76b9719d911017c592<br>Charset: "abcdefghijklmnopqrstuvwxyz"<br>Tamanho: 5<br>Senha: "hello" | 0.004s | 0.005s | 0.004s | 1 |

**O speedup foi linear? Por quê?**
O speedup não foi linear, pois dobrar o número de workers não dobrou a velocidade. Isso acontece porque o trabalho é muito pequeno, e o overhead de criar processos (fork() + execl()) domina o tempo total. Além disso, quando a senha é encontrada cedo, os outros workers terminam rapidamente, reduzindo o benefício da paralelização. Para ter ganhos reais, teria que usar senhas maiores ou charsets maiores, e ajustar o número de workers com o número de núcleos da máquina.

---

## 5. Desafios e Aprendizados
**Qual foi o maior desafio técnico que você enfrentou?**
A maior dificuldade foi descobrir porque senhas numericas davam errado, resolvi quando percebi que estava passando o argumento errado para função `execl`, eu passava `charset_len_str`, mas era para ser `passwor_len_str`.

---

## Comandos de Teste Utilizados

```bash
# Teste básico
./coordinator "900150983cd24fb0d6963f7d28e17f72" 3 "abc" 2

# Teste de performance
time ./coordinator "202cb962ac59075b964b07152d234b70" 3 "0123456789" 1
time ./coordinator "202cb962ac59075b964b07152d234b70" 3 "0123456789" 4

# Teste com senha maior
time ./coordinator "5d41402abc4b2a76b9719d911017c592" 5 "abcdefghijklmnopqrstuvwxyz" 4
```
---

**Checklist de Entrega:**
- [ ] Código compila sem erros
- [ ] Todos os TODOs foram implementados
- [ ] Testes passam no `./tests/simple_test.sh`
- [ ] Relatório preenchido