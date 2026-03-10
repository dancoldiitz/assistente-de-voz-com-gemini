# assistente-de-voz-com-gemini
Criei um assistente de voz com ajuda do Gemini

language = 'pt'

!pip install -q -U google-genai edge-tts

# 1. INSTALAÇÃO DAS DEPENDÊNCIAS
import os
import asyncio
from google import genai
from google.genai import types
from IPython.display import Audio, display, Javascript
from google.colab import output
from base64 import b64decode

# 2. CONFIGURAÇÃO DA API
MINHA_CHAVE = "DIGITE_SUA_CHAVE_API"
client = genai.Client(api_key=MINHA_CHAVE)

# 3. FUNÇÃO DE GRAVAÇÃO
RECORD = """
const sleep = time => new Promise(resolve => setTimeout(resolve, time))
const b2text = blob => new Promise(resolve => {
  const reader = new FileReader()
  reader.onloadend = e => resolve(e.srcElement.result)
  reader.readAsDataURL(blob)
})
var record = time => new Promise(async resolve => {
  stream = await navigator.mediaDevices.getUserMedia({ audio: true })
  recorder = new MediaRecorder(stream)
  chunks = []
  recorder.ondataavailable = e => chunks.push(e.data)
  recorder.start()
  await sleep(time)
  recorder.onstop = async ()=>{
    blob = new Blob(chunks)
    text = await b2text(blob)
    resolve(text)
  }
  recorder.stop()
})
"""

def record(sec=5):
  # Executa o código JavaScript para gravar o áudio
  display(Javascript(RECORD))
  # Recebe o áudio gravado como resultado do JavaScript
  js_result = output.eval_js('record(%s)' % (sec * 1000))
   # Decodifica o áudio em base64
  audio = b64decode(js_result.split(',')[1])
  # Salva o áudio em um arquivo
  file_name = 'request_audio.wav'
  with open(file_name, 'wb') as f:
    f.write(audio)
  # Retorna o caminho do arquivo de áudio (pasta padrão do Google Colab)
  return f'/content/{file_name}'

# Grava o áudio do usuário por um tempo determinado (padrão 5 segundos)
print('Ouvindo...\n')
record_file = record()

# Exibe o áudio gravado
display(Audio(record_file, autoplay=False))
print("Ouvindo... Fale agora!")
audio_path = record(sec=5)

# 4. PROCESSAMENTO DIRETO COM GEMINI
async def sintetizar_voz(texto):
    import edge_tts
    # Usando uma voz brasileira natural
    communicate = edge_tts.Communicate(texto, "pt-BR-AntonioNeural")
    await communicate.save("response_audio.mp3")
    display(Audio("response_audio.mp3", autoplay=True))

# 5. SÍNTESE DE VOZ (RESPOSTA)
async def main():
    try:
        print('🎤 Ouvindo... Fale agora!')
        audio_path = record(sec=5)
        
        with open(audio_path, "rb") as f:
            audio_bytes = f.read()

        print("🧠 Gemini processando áudio...")
        # Usando o modelo mais recente e eficiente
        response = client.models.generate_content(
            model='gemini-2.0-flash',
            contents=[
                types.Part.from_bytes(data=audio_bytes, mime_type='audio/wav'),
                "Responda de forma curta e objetiva em português."
            ]
        )

        gemini_response = response.text
        print(f"\n🤖 Gemini: {gemini_response}")

        # Fala a resposta
        await sintetizar_voz(gemini_response)
        
    except Exception as e:
        print(f"❌ Erro: {e}")

# Inicia o loop
await main()
