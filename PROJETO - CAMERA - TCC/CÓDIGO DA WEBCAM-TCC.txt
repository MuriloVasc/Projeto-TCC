from PIL import Image
import face_recognition
import cv2
import sqlite3

# Função para criar ou conectar ao banco de dados
def connect_db():
    conn = sqlite3.connect('dados_pessoas.db')
    return conn

# Função para criar a tabela de pessoas, se não existir
def criar_tabela():
    conn = connect_db()
    cursor = conn.cursor()
    
    cursor.execute('''CREATE TABLE IF NOT EXISTS pessoas (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        nome TEXT NOT NULL,
        idade INTEGER,
        cpf TEXT,
        ultima_consulta TEXT,
        agendamentos TEXT,
        procedimentos TEXT)''')
    
    conn.commit()
    conn.close()

# Função para buscar dados da pessoa no banco de dados
def buscar_dados(nome):
    conn = connect_db()
    cursor = conn.cursor()
    
    cursor.execute("SELECT * FROM pessoas WHERE nome=?", (nome,))
    dados = cursor.fetchone()
    
    conn.close()
    return dados

# Função para inserir dados de exemplo no banco de dados
def inserir_dados_exemplo():
    conn = connect_db()
    cursor = conn.cursor()
    
    # Exemplo de dados
    dados_exemplo = [
        ("Murilo Gomes", 18, "540.094.658-59", "2024-10-01", "Consulta Geral", "Limpeza")
    ]

    cursor.executemany('''INSERT INTO pessoas (nome, idade, cpf, ultima_consulta, agendamentos, procedimentos) 
    VALUES (?, ?, ?, ?, ?, ?)''', dados_exemplo)
    
    conn.commit()
    conn.close()

# Função principal
def reconhecimento_facial(caminho_imagem):
    # Carregar a imagem de referência
    imagem_referencia = Image.open(caminho_imagem)
    imagem_referencia = imagem_referencia.convert('RGB')
    imagem_referencia_np = face_recognition.load_image_file(caminho_imagem)

    # Tente obter a codificação da imagem de referência
    try:
        face_locations_ref = face_recognition.face_locations(imagem_referencia_np)
        if not face_locations_ref:
            print("Erro: Nenhum rosto encontrado na imagem de referência.")
            return
        encoding_referencia = face_recognition.face_encodings(imagem_referencia_np, face_locations_ref)[0]
    except Exception as e:
        print(f"Erro ao processar a imagem de referência: {e}")
        return

    # Iniciar a captura de vídeo
    video_capture = cv2.VideoCapture(0)
    
    print("Iniciando reconhecimento facial...")

    while True:
        # Capturar um único frame de vídeo
        ret, frame = video_capture.read()
        if not ret:
            print("Erro ao acessar a webcam.")
            break

        # Converter a imagem capturada para o formato RGB
        rgb_frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)

        # Localizar todos os rostos no frame atual do vídeo
        face_locations = face_recognition.face_locations(rgb_frame)

        # Verifique se rostos foram encontrados
        if not face_locations:
            print("Nenhum rosto detectado no frame.")
            continue

        try:
            # Gerar as codificações de face para cada rosto detectado
            face_encodings = face_recognition.face_encodings(rgb_frame, face_locations)
        except Exception as e:
            print(f"Erro ao calcular as codificações das faces: {e}")
            continue

        # Comparar cada rosto detectado com o encoding conhecido
        for (top, right, bottom, left), face_encoding in zip(face_locations, face_encodings):
            matches = face_recognition.compare_faces([encoding_referencia], face_encoding)

            name = "Desconhecido"
            if True in matches:
                # Nome da pessoa deve ser buscado no banco de dados
                dados_pessoa = buscar_dados("Murilo Gomes")  # Exemplo de busca pelo nome
                if dados_pessoa:
                    name = dados_pessoa[1]  # Supondo que o nome está na segunda coluna
                    print("Dados da pessoa:", dados_pessoa)
                else:
                    print("Nenhum dado encontrado para a pessoa reconhecida.")

            # Desenhar um retângulo ao redor do rosto detectado
            cv2.rectangle(frame, (left, top), (right, bottom), (0, 255, 0), 2)
            # Adicionar o nome abaixo do rosto
            cv2.rectangle(frame, (left, bottom - 35), (right, bottom), (0, 255, 0), cv2.FILLED)
            cv2.putText(frame, name, (left + 6, bottom - 6), cv2.FONT_HERSHEY_DUPLEX, 1.0, (255, 255, 255), 1)

        # Exibir o frame com as anotações
        cv2.imshow('Video - Reconhecimento Facial', frame)

        # Pressione 'q' para sair do loop
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break

    # Liberar a captura de vídeo e fechar as janelas
    video_capture.release()
    cv2.destroyAllWindows()

# Criação da tabela e inserção de dados de exemplo
criar_tabela()
inserir_dados_exemplo()

# Execute a função com o caminho da imagem
caminho_imagem = "sua_imagem(0).jpg"  # Substitua pelo caminho correto da imagem
reconhecimento_facial(caminho_imagem)