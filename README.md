
import os
import time
import requests
from smtplib import SMTP, SMTPAuthenticationError
from email.message import EmailMessage
from dotenv import load_dotenv

load_dotenv()

TOKEN = os.getenv("TOKEN_BANXICO")
GMAIL_USER = os.getenv("GMAIL_USER", "oscarhuertamo0408@gmail.com")
GMAIL_TO = os.getenv("GMAIL_TO", GMAIL_USER)
GMAIL_PASS = os.getenv("GMAIL_APP_PASSWORD")

SERIE = "SF60633"
ENLACE_BASE = "https://www.banxico.org.mx/SieAPIRest/service/v1/series"

DIAS = 20
BASE_DIAS = 360

estado = {
    "tasa": None,
    "fecha": None,
    "capital": None,
    "interes": None,
    "texto": None,
}

def banxico_cetes() -> tuple[float, str]:
    if not TOKEN:
        raise ValueError("No se encontró TOKEN_BANXICO en el archivo .env")

    headers = {"Bmx-Token": TOKEN.strip()}
    enlace = f"{ENLACE_BASE}/{SERIE}/datos/oportuno"
    response = requests.get(url=enlace, headers=headers, timeout=15)

    if not response.ok:
        raise RuntimeError(f"Banxico respondió {response.status_code}: {response.text[:200]}")

    payload = response.json()
    dato = payload["bmx"]["series"][0]["datos"][0]

    return float(dato["dato"]), dato["fecha"]

def calcular_isc(capital: float, tasa_anual_pct: float, dias: int = DIAS) -> float:
    return capital * (tasa_anual_pct / 100) * (dias / BASE_DIAS)

def enviar_correo(asunto: str, cuerpo: str) -> None:
    if not GMAIL_PASS:
        raise ValueError("Falta GMAIL_APP_PASSWORD en el .env")

    msg = EmailMessage()
    msg["Subject"] = asunto
    msg["From"] = GMAIL_USER
    msg["To"] = GMAIL_TO
    msg.set_content(cuerpo)

    try:
        with SMTP("smtp.gmail.com", 587, timeout=20) as smtp:
            smtp.starttls()
            smtp.login(GMAIL_USER, GMAIL_PASS.replace(" ", ""))
            smtp.send_message(msg)
    except SMTPAuthenticationError:
        raise RuntimeError(
            "Gmail rechazó el login. Usa una contraseña de aplicación "
            "con verificación en dos pasos activada."
        )

def enviar_respaldo(texto: str) -> None:
    print("[Respaldo SMS/WhatsApp]")
    print(texto)

def menu_principal() -> None:
    while True:
        print("\nFlujo de Programa")
        print("1. Consulta Banxico CETES a 28 días")
        print("2. Calcular ISC a 20 días")
        print("3. Enviar resultados por correo")
        print("4. Probar respaldo SMS/WhatsApp")
        print("5. Salir")

        opcion = input("Elige una opción (1-5): ").strip()

        if opcion == "1":
            try:
                estado["tasa"], estado["fecha"] = banxico_cetes()
                print(f"Tasa CETES ({SERIE}) al {estado['fecha']}: {estado['tasa']:.2f}% anual")
            except Exception as e:
                print(f"\n[Error] No se pudo consultar Banxico: {e}")
            time.sleep(1.2)

        elif opcion == "2":
            if estado["tasa"] is None:
                print("\n[Aviso] Primero consulta la tasa con la opción 1.")
            else:
                try:
                    capital = float(input("Capital a invertir: ").replace(",", "").strip())
                except ValueError:
                    print("\n[Error] Ingresa un número válido.")
                    time.sleep(1.2)
                    continue

                interes = calcular_isc(capital, estado["tasa"], DIAS)
                estado["capital"] = capital
                estado["interes"] = interes
                estado["texto"] = (
                    f"Tasa CETES ({SERIE}) al {estado['fecha']}: {estado['tasa']:.2f}% anual\n"
                    f"Capital: ${capital:,.2f}\n"
                    f"Plazo: {DIAS} días (base {BASE_DIAS})\n"
                    f"Interés simple: ${interes:,.2f}\n"
                    f"Monto final: ${capital + interes:,.2f}"
                )
                print("\n" + estado["texto"])
            time.sleep(1.2)

        elif opcion == "3":
            if estado["texto"] is None:
                print("\n[Aviso] Primero consulta la tasa (1) y calcula el ISC (2).")
            else:
                try:
                    enviar_correo("Interés simple CETES a 20 días", estado["texto"])
                    print("\nCorreo enviado correctamente.")
                except Exception as e:
                    print(f"\nNo se pudo enviar el correo: {e}")
                    print("Activando respaldo por SMS/WhatsApp...")
                    enviar_respaldo(estado["texto"])
            time.sleep(1.2)

        elif opcion == "4":
            texto = estado["texto"] or "Prueba de respaldo: CETES interés simple."
            enviar_respaldo(texto)
            time.sleep(1.2)

        elif opcion == "5":
            print("\nCerrando el sistema... ¡Hasta luego!")
            break

        else:
            print("\n[Error] Opción no válida. Ingresa un número del 1 al 5.")

if __name__ == "__main__":
    menu_principal()
