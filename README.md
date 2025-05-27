import streamlit as st
import gspread
from oauth2client.service_account import ServiceAccountCredentials
from datetime import datetime
import pandas as pd # Aunque no mostremos el DataFrame global, gspread puede usarlo internamente o podrías quererlo para otras cosas. Si no, se puede quitar.
import time

# --- CONFIGURACIÓN INICIAL Y CONSTANTES ---
KEYFILE_NAME = "parcial-459600-b0006697b857.json"
GOOGLE_SHEET_NAME = "Asistencia"

TARGET_HEADERS_CONFIG = {
    "id": "IDENTIFICACION",
    "name": "APELLIDOS Y NOMBRES",
    "saldo_mora": "SALDO EN MORA"
}
MAX_ROWS_TO_SCAN_FOR_HEADERS = 10 # Cuántas filas escanear para encontrar los encabezados
MAX_RETRIES_SHEET_READ = 3
INITIAL_BACKOFF_SECONDS = 2

scope = ["https://spreadsheets.google.com/feeds", "https://www.googleapis.com/auth/drive"]

# --- AUTENTICACIÓN Y CLIENTE GSPREAD (CACHEADO) ---
@st.cache_resource
def get_gspread_libro(keyfile_path, sheet_name_to_open):
    try:
        creds = ServiceAccountCredentials.from_json_keyfile_name(keyfile_path, scope)
        client = gspread.authorize(creds)
        libro_obj = client.open(sheet_name_to_open)
        print(f"INFO (get_gspread_libro): Cliente GSpread y Libro '{sheet_name_to_open}' Autorizado/Abierto.") # Backend log
        return libro_obj
    except FileNotFoundError:
        st.error(f"⚠️ Error Crítico: No se encontró el archivo de credenciales '{keyfile_path}'. La aplicación no puede funcionar.")
        st.stop()
    except gspread.exceptions.SpreadsheetNotFound:
        st.error(f"⚠️ Error Crítico: No se encontró el libro '{sheet_name_to_open}'. Verifica el nombre y los permisos. La aplicación no puede funcionar.")
        st.stop()
    except Exception as e:
        st.error(f"⚠️ Error Crítico durante la autenticación o apertura del libro: {e}. La aplicación no puede funcionar.")
        st.stop()

libro = get_gspread_libro(KEYFILE_NAME, GOOGLE_SHEET_NAME)
# No es necesario un st.success aquí para una interfaz más limpia si solo tenemos una funcionalidad.

# --- FUNCIONES AUXILIARES (NECESARIAS PARA EL REGISTRO) ---
def get_column_indices(header_row_data, target_headers_map):
    indices = {}
    if not header_row_data:
        for key in target_headers_map: indices[key] = None
        return indices
    header_row_cleaned = [str(h).strip().upper() for h in header_row_data]
    for key, target_name in target_headers_map.items():
        try: indices[key] = header_row_cleaned.index(target_name.upper())
        except ValueError: indices[key] = None
    return indices

def fetch_sheet_data_with_retries(sheet_obj):
    data = None
    for attempt in range(MAX_RETRIES_SHEET_READ):
        try:
            data = sheet_obj.get_all_values()
            break
        except gspread.exceptions.APIError as api_e:
            if api_e.response.status_code == 429:
                wait_time = INITIAL_BACKOFF_SECONDS * (2**attempt)
                print(f"DEBUG (fetch_sheet_data): Cuota API para '{sheet_obj.title}'. Reintento {attempt+1} en {wait_time}s") # Backend log
                time.sleep(wait_time)
                if attempt == MAX_RETRIES_SHEET_READ - 1: raise
            else: raise
    return data

def get_person_details_by_id(g_libro, cedula_to_find, headers_map, max_scan_rows):
    print(f"INFO (get_person_details_by_id): Buscando detalles para cédula: {cedula_to_find}") # Backend log
    try:
        all_sheets_for_lookup = g_libro.worksheets()
    except Exception as e:
        print(f"ERROR (get_person_details_by_id): Error obteniendo worksheets: {e}") # Backend log
        return None

    for sheet_obj in all_sheets_for_lookup:
        if sheet_obj.title.lower() in ["morosos", "asistencia"]: # No buscar datos de personas aquí
            continue
        
        try:
            sheet_data = fetch_sheet_data_with_retries(sheet_obj)
            if not sheet_data: continue

            header_idx = -1
            col_idxs = None
            for r_idx, current_row in enumerate(sheet_data[:max_scan_rows]):
                temp_idxs = get_column_indices(current_row, headers_map)
                if temp_idxs.get("id") is not None and temp_idxs.get("name") is not None and temp_idxs.get("saldo_mora") is not None:
                    header_idx = r_idx
                    col_idxs = temp_idxs
                    break
            
            if header_idx != -1:
                data_start_idx = header_idx + 1
                for row_data in sheet_data[data_start_idx:]:
                    try:
                        current_id = ""
                        # Asegurarse de que el índice de 'id' no es None y está dentro de los límites de la fila
                        if col_idxs.get("id") is not None and col_idxs["id"] < len(row_data):
                           current_id = str(row_data[col_idxs["id"]]).strip()
                        
                        if current_id == cedula_to_find:
                            name_val = "Nombre no encontrado"
                            if col_idxs.get("name") is not None and col_idxs["name"] < len(row_data):
                                name_val = str(row_data[col_idxs["name"]]).strip()
                            
                            saldo_val = "Saldo no especificado"
                            if col_idxs.get("saldo_mora") is not None and col_idxs["saldo_mora"] < len(row_data):
                                saldo_val = str(row_data[col_idxs["saldo_mora"]]).strip()
                            print(f"INFO (get_person_details_by_id): Detalles encontrados para {cedula_to_find}: {name_val}, {saldo_val} en {sheet_obj.title}") # Backend log
                            return {"name": name_val, "saldo_mora": saldo_val, "sheet_name": sheet_obj.title}
                    except IndexError:
                        continue 
        except Exception as e_detail_sheet:
            print(f"WARN (get_person_details_by_id): Error procesando hoja '{sheet_obj.title}' durante búsqueda de detalles: {e_detail_sheet}") # Backend log
            continue
    print(f"INFO (get_person_details_by_id): No se encontraron detalles para la cédula {cedula_to_find} en ninguna hoja de datos.") # Backend log
    return None


# --- SECCIÓN: REGISTRO DE ASISTENCIA INDIVIDUAL (ÚNICA SECCIÓN VISIBLE) ---
st.title("📋 Registro de Asistencia Individual") # Título principal de la página
st.markdown("Por favor, ingrese su número de documento para registrar su asistencia.")

cedula_registro_input = st.text_input("🔑 Número de Documento:", key="cedula_input_asistencia")

if st.button("✅ Validar y Registrar Asistencia"):
    cedula_limpia_registro = cedula_registro_input.strip()
    if not cedula_limpia_registro:
        st.warning("Debe ingresar un número de documento.")
    else:
        try:
            # 1. Verificar si la cédula existe en los registros de datos generales y obtener detalles
            person_details_for_registration = get_person_details_by_id(libro, cedula_limpia_registro, TARGET_HEADERS_CONFIG, MAX_ROWS_TO_SCAN_FOR_HEADERS)
            
            if not person_details_for_registration:
                st.error(f"🚫 Cédula {cedula_limpia_registro} no encontrada en los registros generales. No se puede registrar asistencia.", icon="🚫")
                st.stop() # Detener el proceso aquí

            # 2. Verificar si ya está registrado en la hoja "asistencia"
            asistencia_sheet_title = "asistencia"
            try:
                sheet_asistencia_check = libro.worksheet(asistencia_sheet_title)
                asistencias_registradas = sheet_asistencia_check.col_values(1) # Asume cédulas en Col A
                if cedula_limpia_registro in [val.strip() for val in asistencias_registradas if val.strip()]:
                    # Usar el nombre obtenido de person_details_for_registration
                    nombre_persona = person_details_for_registration.get('name', cedula_limpia_registro)
                    st.error(f"🚫 {nombre_persona} (Cédula: {cedula_limpia_registro}) ya se registró anteriormente.", icon="🚫")
                    st.stop() 
            except gspread.exceptions.WorksheetNotFound:
                # La hoja "asistencia" no existe aún, lo cual es normal la primera vez. Continuar.
                pass 
            except Exception as e_check_asist:
                st.error(f"⚠️ Error verificando registros de asistencia previos: {e_check_asist}")
                st.stop()

            # 3. Validar contra la hoja 'morosos'
            morosos_sheet_title = "morosos"
            cedulas_morosas_para_registro_set = set() # Siempre obtenerla fresca para esta validación crítica
            try:
                sheet_morosos_val_reg = libro.worksheet(morosos_sheet_title)
                valores_morosos_val_reg = sheet_morosos_val_reg.col_values(1)
                for val in valores_morosos_val_reg:
                    if val.strip(): cedulas_morosas_para_registro_set.add(val.strip())
            except gspread.exceptions.WorksheetNotFound:
                st.error(f"⚠️ No se encontró la hoja '{morosos_sheet_title}'. No se puede validar el estado de mora para el registro.")
                st.stop()
            except Exception as e_mor_reg:
                st.error(f"⚠️ Error accediendo a la hoja '{morosos_sheet_title}' para validación: {e_mor_reg}")
                st.stop()

            if cedula_limpia_registro in cedulas_morosas_para_registro_set:
                # Usamos person_details_for_registration que ya obtuvimos
                nombre_persona = person_details_for_registration.get('name', cedula_limpia_registro)
                saldo_mora_persona = person_details_for_registration.get('saldo_mora', 'No especificado')
                st.error(f"❌ {nombre_persona} (Cédula: {cedula_limpia_registro}): Tiene una mora pendiente de {saldo_mora_persona}. No puede registrar asistencia.", icon="❌")
            else:
                # 4. Si todas las validaciones pasan, proceder a registrar en la hoja 'asistencia'
                sheet_asistencia_reg = None
                try:
                    sheet_asistencia_reg = libro.worksheet(asistencia_sheet_title)
                except gspread.exceptions.WorksheetNotFound:
                    st.info(f"Hoja '{asistencia_sheet_title}' no encontrada. Creándola ahora...", icon="✍️")
                    try:
                        sheet_asistencia_reg = libro.add_worksheet(title=asistencia_sheet_title, rows="100", cols="2") # Ajusta rows/cols según necesites
                        sheet_asistencia_reg.append_row(["Documento", "Fecha y Hora Registro"]) # Encabezados opcionales
                        st.success(f"Hoja '{asistencia_sheet_title}' creada exitosamente.")
                    except Exception as e_create_sheet:
                        st.error(f"⚠️ No se pudo crear la hoja '{asistencia_sheet_title}': {e_create_sheet}")
                        st.stop() 
                except Exception as e_asist_open:
                     st.error(f"⚠️ Error accediendo a la hoja '{asistencia_sheet_title}': {e_asist_open}")
                     st.stop()

                now_str = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
                sheet_asistencia_reg.append_row([cedula_limpia_registro, now_str])
                # Usamos person_details_for_registration para el mensaje de éxito
                nombre_persona = person_details_for_registration.get('name', cedula_limpia_registro)
                st.success(f"🟢 Asistencia registrada para: {nombre_persona} (Cédula: {cedula_limpia_registro}) a las {now_str}.", icon="🎉")

        except Exception as e_reg_general: # Captura cualquier otra excepción durante el proceso de registro
            st.error(f"⚠️ Error en el proceso de registro de asistencia: {e_reg_general}")
            # import traceback # Descomentar para depuración más profunda si es necesario
            # st.error(traceback.format_exc())
