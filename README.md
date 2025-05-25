import json
import os
import hashlib
from datetime import datetime

ARQUIVO_USUARIOS = "usuarios.json"

# ----------------- UTILITÁRIOS ------------------

def gerar_hash(senha):
    return hashlib.sha256(senha.encode()).hexdigest()

def validar_email(email):
    return "@" in email and "." in email

def carregar_usuarios():
    if os.path.exists(ARQUIVO_USUARIOS):
        with open(ARQUIVO_USUARIOS, "r", encoding="utf-8") as f:
            return json.load(f)
    return []

def salvar_usuarios(usuarios):
    with open(ARQUIVO_USUARIOS, "w", encoding="utf-8") as f:
        json.dump(usuarios, f, indent=4)

def carregar_conteudo_por_nivel(nivel):
    arquivo = f"{nivel}.json"
    if os.path.exists(arquivo):
        with open(arquivo, "r", encoding="utf-8") as f:
            return json.load(f)
    return {}

def carregar_quiz(nivel):
    arquivo = f"{nivel}_quiz.json"
    if os.path.exists(arquivo):
        with open(arquivo, "r", encoding="utf-8") as f:
            return json.load(f)
    return []

def registrar_desempenho(email, nivel, acertos, total):
    usuarios = carregar_usuarios()
    for usuario in usuarios:
        if usuario["email"] == email:
            usuario.setdefault("desempenho", []).append({
                "data": datetime.now().strftime("%Y-%m-%d"),
                "nivel": nivel,
                "acertos": acertos,
                "total": total
            })
            break
    salvar_usuarios(usuarios)

def exibir_estatisticas(usuario_email):
    usuarios = carregar_usuarios()
    for usuario in usuarios:
        if usuario["email"] == usuario_email and "desempenho" in usuario:
            desempenhos = usuario["desempenho"]
            total_quizzes = len(desempenhos)
            total_acertos = sum(d["acertos"] for d in desempenhos)
            total_perguntas = sum(d["total"] for d in desempenhos)
            media_acertos = total_acertos / total_quizzes if total_quizzes > 0 else 0

            print(f"\nEstatísticas de desempenho para {usuario['nome']}:")
            print(f"- Total de quizzes: {total_quizzes}")
            print(f"- Média de acertos: {media_acertos:.2f}")
            print(f"- Total de acertos: {total_acertos} / {total_perguntas}")

            melhor = max(desempenhos, key=lambda d: d["acertos"])
            pior = min(desempenhos, key=lambda d: d["acertos"])

            print(f"- Melhor desempenho: {melhor['acertos']} de {melhor['total']} em {melhor['data']}")
            print(f"- Pior desempenho: {pior['acertos']} de {pior['total']} em {pior['data']}")
            return

    print("Nenhum dado de desempenho encontrado.")

# ----------------- JOGO DA VELHA ------------------

def jogo_da_velha():
    tabuleiro = [" "] * 9

    def mostrar_tabuleiro():
        print()
        print(f" {tabuleiro[0]} | {tabuleiro[1]} | {tabuleiro[2]} ")
        print("---+---+---")
        print(f" {tabuleiro[3]} | {tabuleiro[4]} | {tabuleiro[5]} ")
        print("---+---+---")
        print(f" {tabuleiro[6]} | {tabuleiro[7]} | {tabuleiro[8]} ")
        print()

    def checar_vitoria(jogador):
        combinacoes = [
            (0, 1, 2), (3, 4, 5), (6, 7, 8),
            (0, 3, 6), (1, 4, 7), (2, 5, 8),
            (0, 4, 8), (2, 4, 6)
        ]
        return any(all(tabuleiro[pos] == jogador for pos in combo) for combo in combinacoes)

    def checar_empate():
        return all(casa != " " for casa in tabuleiro)

    jogador = "X"
    print("Bem-vindo ao Jogo da Velha!")
    mostrar_tabuleiro()

    while True:
        try:
            jogada = int(input(f"Jogador {jogador}, escolha uma posição (1-9): ")) - 1
            if jogada not in range(9):
                print("Posição inválida.")
                continue
            if tabuleiro[jogada] != " ":
                print("Posição ocupada.")
                continue
            tabuleiro[jogada] = jogador
            mostrar_tabuleiro()

            if checar_vitoria(jogador):
                print(f"Parabéns! Jogador {jogador} venceu!")
                break
            if checar_empate():
                print("Empate!")
                break

            jogador = "O" if jogador == "X" else "X"
        except ValueError:
            print("Entrada inválida. Use números de 1 a 9.")

    input("Pressione Enter para voltar ao menu...")

# ----------------- SISTEMA DE USUÁRIOS ------------------

def cadastrar_usuario():
    print("Cadastro de novo usuário")
    nome = input("Nome completo: ").strip()
    email = input("E-mail: ").strip()

    if not validar_email(email):
        print("E-mail inválido.")
        return

    senha = input("Crie uma senha: ").strip()
    confirmar = input("Confirme sua senha: ").strip()

    if senha != confirmar:
        print("As senhas não coincidem.")
        return

    print("Níveis disponíveis: iniciante, intermediario, avançado")
    nivel = input("Nível de conhecimento: ").strip().lower()
    if nivel not in ["iniciante", "intermediario", "avançado"]:
        print("Nível inválido.")
        return

    consentimento = input("Autoriza uso dos dados para fins educacionais? (s/n): ").lower()
    if consentimento != "s":
        print("Consentimento negado.")
        return

    usuarios = carregar_usuarios()
    if any(u["email"] == email for u in usuarios):
        print("E-mail já cadastrado.")
        return

    novo_usuario = {
        "nome": nome,
        "email": email,
        "senha": gerar_hash(senha),
        "nivel": nivel,
        "desempenho": []
    }

    usuarios.append(novo_usuario)
    salvar_usuarios(usuarios)
    print(f"Usuário '{nome}' cadastrado com sucesso!")

def fazer_login():
    print("Login de usuário")
    email = input("E-mail: ").strip()
    senha = input("Senha: ").strip()
    senha_hash = gerar_hash(senha)

    usuarios = carregar_usuarios()
    for usuario in usuarios:
        if usuario["email"] == email and usuario["senha"] == senha_hash:
            print(f"\nLogin realizado! Bem-vindo(a), {usuario['nome']} ({usuario['nivel']})")
            menu_conteudos(usuario)
            return
    print("E-mail ou senha incorretos.")

# ----------------- QUIZ ------------------

def executar_quiz(nivel, email_usuario):
    perguntas = carregar_quiz(nivel)
    if not perguntas:
        print("Nenhum quiz disponível para este nível.")
        return

    print(f"\nQuiz - Nível: {nivel.capitalize()}")
    acertos = 0

    for i, q in enumerate(perguntas, 1):
        print(f"\n{i}. {q['pergunta']}")
        for idx, opcao in enumerate(q["opcoes"]):
            print(f"  {idx + 1}. {opcao}")
        resposta = input("Sua resposta (número): ").strip()

        if resposta.isdigit() and int(resposta) - 1 == q["resposta_correta"]:
            print("Correto!")
            acertos += 1
        else:
            correta = q["opcoes"][q["resposta_correta"]]
            print(f"Errado. A resposta correta é: {correta}")

    print(f"\nVocê acertou {acertos} de {len(perguntas)} questões.")
    registrar_desempenho(email_usuario, nivel, acertos, len(perguntas))
    input("Pressione Enter para voltar...")

# ----------------- MENU DE CONTEÚDO ------------------

def menu_conteudos(usuario):
    nivel = usuario["nivel"]
    conteudos = carregar_conteudo_por_nivel(nivel)

    if not conteudos:
        print("Conteúdo indisponível.")
        return

    while True:
        print(f"\nConteúdos - Nível: {nivel.capitalize()}")
        temas = list(conteudos.keys())

        for i, tema in enumerate(temas, 1):
            print(f"{i}. {tema}")
        print(f"{len(temas)+1}. Fazer Quiz")
        print(f"{len(temas)+2}. Jogar Jogo da Velha")
        print(f"{len(temas)+3}. Ver Estatísticas")
        print(f"{len(temas)+4}. Sair")

        opcao = input("Escolha uma opção: ").strip()

        if opcao.isdigit():
            idx = int(opcao)
            if 1 <= idx <= len(temas):
                tema = temas[idx - 1]
                print(f"\n{tema}:\n{conteudos[tema]}")
                input("\nPressione Enter para voltar...")
            elif idx == len(temas) + 1:
                executar_quiz(nivel, usuario["email"])
            elif idx == len(temas) + 2:
                jogo_da_velha()
            elif idx == len(temas) + 3:
                exibir_estatisticas(usuario["email"])
            elif idx == len(temas) + 4:
                print("Saindo do menu de conteúdos.")
                break
            else:
                print("Opção inválida.")
        else:
            print("Digite apenas o número da opção.")

# ----------------- MENU PRINCIPAL ------------------

def menu():
    while True:
        print("\n==== Plataforma de Inclusão Tecnológica ====")
        print("1. Cadastrar novo usuário")
        print("2. Fazer login e acessar conteúdos")
        print("3. Sair")

        escolha = input("Escolha uma opção: ").strip()

        if escolha == "1":
            cadastrar_usuario()
        elif escolha == "2":
            fazer_login()
        elif escolha == "3":
            print("Encerrando...")
            break
        else:
            print("Opção inválida. Tente novamente.")

if __name__ == "__main__":
    menu()
