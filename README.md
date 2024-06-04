import streamlit as st
import azure.cognitiveservices.speech as speechsdk
import threading
import time
import requests

# Azure Cognitive Services subscription keys (replace with your own)
speech_key = "332153544068453e9b2036518ed8ab83"
service_region = "centralindia"
translator_key = "acdc1c81e2974ae284b5ec52e9f0bfb"
translator_endpoint = "https://api.cognitive.microsofttranslator.com/"
translator_location = "centralindia"

# Buffer to store recognized text
recognized_text_buffer = []

def translate_text(input_language, output_language, text):
    path = '/translate?api-version=3.0'
    params = f'&from={input_language}&to={output_language}'
    constructed_url = translator_endpoint + path + params

    headers = {
        'Ocp-Apim-Subscription-Key': translator_key,
        'Ocp-Apim-Subscription-Region': translator_location,
        'Content-type': 'application/json'
    }

    body = [{'text': text}]

    try:
        request = requests.post(constructed_url, headers=headers, json=body)
        response = request.json()

        # Print the full API response for debugging (optional)
        print("Translation response:", response)

        translated_text = response[0]['translations'][0]['text']
        return translated_text
    except Exception as e:
        print("Error occurred during translation:", e)
        return "Translation error"

def synthesize_speech(output_language, text):
    speech_config = speechsdk.SpeechConfig(subscription=speech_key, region=service_region)

    # Set the appropriate voice for the output language
    voice_name = "si-LK-ThiliniNeural" if output_language == "si-LK" else "en-US-AvaMultilingualNeural"
    speech_config.speech_synthesis_voice_name = voice_name

    # Synthesize the translated text to speech
    speech_synthesizer = speechsdk.SpeechSynthesizer(speech_config=speech_config)
    synth_result = speech_synthesizer.speak_text_async(text).get()

    if synth_result.reason == speechsdk.ResultReason.SynthesizingAudioCompleted:
        print(f"Speech synthesized for text [{text}]")
    else:
        print(f"Speech synthesis canceled, {synth_result.cancellation_details.reason}")

def recognize_from_microphone_continuous(input_language, output_language):
    speech_config = speechsdk.SpeechConfig(subscription=speech_key, region=service_region)
    speech_config.speech_recognition_language = input_language

    audio_config = speechsdk.audio.AudioConfig(use_default_microphone=True)
    speech_recognizer = speechsdk.SpeechRecognizer(speech_config=speech_config, audio_config=audio_config)

    def on_recognized(event_args):
        if event_args.result.reason == speechsdk.ResultReason.RecognizedSpeech:
            recognized_text = event_args.result.text
            recognized_text_buffer.append(recognized_text)
            print(f"Recognized: {recognized_text}")

    speech_recognizer.recognized.connect(on_recognized)

    print("Speak into your microphone. Press Ctrl+C to stop.")
    speech_recognizer.start_continuous_recognition()

    try:
        while True:
            time.sleep(0.1)
    except KeyboardInterrupt:
        print("Stopping recognition...")
        speech_recognizer.stop_continuous_recognition()
        full_recognized_text = ' '.join(recognized_text_buffer).strip()
        print(f"Full recognized text: {full_recognized_text}")
        if full_recognized_text:
            translated_text = translate_text(input_language, output_language, full_recognized_text)
            st.write(f"Translated Text ({output_language}): {translated_text}")
            synthesize_speech(output_language, translated_text)
        else:
            print("No text recognized to translate.")

def main():
    """
    Streamlit chatbot for Sinhala-English translation.
    """

    st.title
