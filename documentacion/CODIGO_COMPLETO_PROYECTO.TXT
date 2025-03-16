import streamlit as st
import sqlite3
import pandas as pd
import datetime
from io import BytesIO
import os
import requests
from bs4 import BeautifulSoup
import re
import base64
import webbrowser

# --- Estilos y JavaScript ---
st.markdown("""
    <style>
    .stApp {
        max-width: 1200px;
        margin: auto;
    }
    input, textarea {
        font-size: 1.1em;
    }
    </style>
    """, unsafe_allow_html=True)

disable_enter_js = """
<script>
document.addEventListener('DOMContentLoaded', function() {
  const inputs = document.querySelectorAll('input');
  inputs.forEach(input => {
    input.addEventListener('keydown', function(e) {
      if (e.key === 'Enter') {
        e.preventDefault();
      }
    });
  });
});
</script>
"""

# --- Función de Scraping (Extraída de COD_EXTRACCION.txt) ---
def scrape_vehicle_details(url):
    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/98.0.4758.102 Safari/537.36',
        'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
        'Accept-Language': 'es-ES,es;q=0.9,en-US;q=0.8,en;q=0.7',
        'Referer': 'https://www.chileautos.cl/'
    }
    
    response = requests.get(url, headers=headers)
    if response.status_code != 200:
        st.error(f"Error al obtener la página: {response.status_code}")
        return None

    soup = BeautifulSoup(response.content, "html.parser")

    # --- Extracción de la imagen de contacto ---
    contact_img_tag = soup.find("img", src=lambda src: src and src.startswith("data:image"))
    if contact_img_tag:
        img_src = contact_img_tag.get("src", "")
        if "base64," in img_src:
            base64_data = img_src.split("base64,", 1)[1].strip()
            base64_data = "".join(base64_data.split())
            try:
                image_bytes = base64.b64decode(base64_data)
                with open("contact_image.png", "wb") as f:
                    f.write(image_bytes)
                contact_image_file = "contact_image.png"
            except Exception as e:
                st.error("Error al decodificar la imagen: " + str(e))
                contact_image_file = "Error al decodificar"
        else:
            contact_image_file = "Formato de imagen no reconocido"
    else:
        contact_image_file = "No encontrado"

    # --- Extracción de información del vehículo (nombre, año, precio) ---
    nombre, anio, precio = None, None, None

    vehiculo_elem = soup.find("div", class_="features-item-value-vehculo")
    if vehiculo_elem:
        texto_vehiculo = vehiculo_elem.get_text(strip=True)
        partes = texto_vehiculo.split(" ", 1)
        if partes and partes[0].isdigit() and len(partes[0]) == 4:
            anio = partes[0]
            nombre = partes[1] if len(partes) > 1 else ""
        else:
            nombre = texto_vehiculo

    if not nombre:
        h1_elem = soup.find("h1")
        if h1_elem:
            titulo_texto = h1_elem.get_text(strip=True)
            partes = titulo_texto.split(" ", 1)
            if partes and partes[0].isdigit() and len(partes[0]) == 4:
                anio = partes[0]
                nombre = partes[1] if len(partes) > 1 else titulo_texto
            else:
                nombre = titulo_texto

    if anio:
        nombre_completo = f"{anio} {nombre}"
    else:
        nombre_completo = nombre

    # --- Extracción del precio ---
    precio_elem = soup.find("div", class_="features-item-value-precio")
    if precio_elem:
        precio_texto = precio_elem.get_text(strip=True)
        match = re.search(r"\$(\d{1,3}(?:,\d{3})+)", precio_texto)
        if match:
            precio = match.group(1)
        else:
            precio = precio_texto

    # --- Extracción de la descripción ---
    descripcion = None
    descripcion_container = soup.find("div", class_="view-more-container")
    if descripcion_container:
        view_more_target = descripcion_container.find("div", class_="view-more-target")
        if view_more_target:
            p_elem = view_more_target.find("p")
            if p_elem:
                descripcion = p_elem.get_text(strip=True)
    if not descripcion:
        descripcion = "No disponible"

    return {
        "nombre": nombre_completo if nombre_completo else "No disponible",
        "anio": anio if anio else "No disponible",
        "precio": precio if precio else "No disponible",
        "descripcion": descripcion,
        "contact_image_file": contact_image_file
    }

# --- Callback para actualizar los campos del auto ---
def update_auto_fields():
    url = st.session_state.link_auto  # Se usa el valor del campo fuera del formulario
    if url:
        data = scrape_vehicle_details(url.strip())
        if data:
            st.session_state.auto_modelo = data['nombre']
            st.session_state.precio_str = data['precio']
            st.session_state.descripcion_contacto = data['descripcion']

# --- Configuración de la Base de Datos ---
db_filename = 'datos_consignacion.db'

def get_connection():
    conn = sqlite3.connect(db_filename, check_same_thread=False)
    return conn

def create_tables(conn):
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS links_contactos (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            link_general TEXT NOT NULL,
            fecha_creacion TEXT NOT NULL,
            marca TEXT NOT NULL,
            descripcion TEXT NOT NULL
        )
    ''')
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS contactos (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            link_auto TEXT NOT NULL,
            telefono TEXT UNIQUE NOT NULL,
            nombre TEXT NOT NULL,
            auto TEXT NOT NULL,
            precio REAL NOT NULL,
            descripcion TEXT NOT NULL,
            id_link INTEGER,
            FOREIGN KEY (id_link) REFERENCES links_contactos(id)
        )
    ''')
    conn.commit()

conn = get_connection()
create_tables(conn)

# --- Navegación en Streamlit con botones verticales ---
if 'page' not in st.session_state:
    st.session_state.page = "Crear Link Contactos"

st.sidebar.title("Navegación")
if st.sidebar.button("Crear Link Contactos"):
    st.session_state.page = "Crear Link Contactos"
if st.sidebar.button("Agregar Contactos"):
    st.session_state.page = "Agregar Contactos"
if st.sidebar.button("Ver Contactos & Exportar"):
    st.session_state.page = "Ver Contactos & Exportar"

page = st.session_state.page

# --- Página: Crear Link Contactos ---
if page == "Crear Link Contactos":
    st.title("Crear Link Contactos")
    st.markdown(disable_enter_js, unsafe_allow_html=True)
    with st.form("crear_link_form"):
        link_general = st.text_input("Link General")
        fecha_creacion = st.date_input("Fecha de Creación", value=datetime.date.today())
        marca = st.text_input("Marca")
        descripcion = st.text_area("Descripción")
        submitted = st.form_submit_button("Crear Link")
    if submitted:
        if not link_general.strip() or not marca.strip() or not descripcion.strip():
            st.error("Todos los campos son requeridos.")
        else:
            cursor = conn.cursor()
            cursor.execute('''
                INSERT INTO links_contactos (link_general, fecha_creacion, marca, descripcion)
                VALUES (?, ?, ?, ?)
            ''', (link_general.strip(), fecha_creacion.strftime("%Y-%m-%d"), marca.strip(), descripcion.strip()))
            conn.commit()
            st.success("Link Contactos creado exitosamente.")

# --- Página: Agregar Contactos ---
elif page == "Agregar Contactos":
    st.title("Agregar Contactos")
    st.markdown(disable_enter_js, unsafe_allow_html=True)
    
    df_links = pd.read_sql_query("SELECT * FROM links_contactos", conn)
    if df_links.empty:
        st.warning("No existen links. Por favor, crea un Link Contactos primero.")
    else:
        df_links['display'] = df_links.apply(
            lambda row: f"{row['link_general']} - {row['marca']}", axis=1
        )
        opcion = st.selectbox("Selecciona el Link Contactos", df_links['display'])
        selected_link = df_links[df_links['display'] == opcion].iloc[0]
        st.markdown(f"**Fecha de Creación:** {selected_link['fecha_creacion']}")
        st.markdown(f"**Marca:** {selected_link['marca']}")
        st.markdown(f"**Descripción del Link:** {selected_link['descripcion']}")
        
        link_id = selected_link["id"]
        
        # --- Campo de Link del Auto fuera del formulario para activar el callback ---
        st.text_input("Link del Auto", key="link_auto", on_change=update_auto_fields)
        
        with st.form("agregar_contacto_form"):
            # Los campos se pre-cargan con la información extraída, si existe.
            telefono = st.text_input("Teléfono")
            nombre = st.text_input("Nombre")
            auto_modelo = st.text_input("Auto", key="auto_modelo")
            precio_str = st.text_input("Precio (ej: 10,500,000)", key="precio_str")
            descripcion_contacto = st.text_area("Descripción del Contacto", key="descripcion_contacto")
            submitted_contacto = st.form_submit_button("Agregar Contacto")
        
        if submitted_contacto:
            if (not st.session_state.get("link_auto", "").strip() or not telefono.strip() or 
                not auto_modelo.strip() or not precio_str.strip() or not descripcion_contacto.strip()):
                st.error("Todos los campos son requeridos.")
            else:
                try:
                    precio_clean = precio_str.replace(",", "").strip()
                    precio = float(precio_clean)
                except ValueError:
                    st.error("Precio inválido. Asegúrate de ingresar un número, por ejemplo: 10,500,000")
                    st.stop()
                try:
                    cursor = conn.cursor()
                    cursor.execute('''
                        INSERT INTO contactos (link_auto, telefono, nombre, auto, precio, descripcion, id_link)
                        VALUES (?, ?, ?, ?, ?, ?, ?)
                    ''', (st.session_state.link_auto.strip(), telefono.strip(), nombre.strip(), auto_modelo.strip(), precio, descripcion_contacto.strip(), link_id))
                    conn.commit()
                    st.success("Contacto agregado exitosamente.")
                except sqlite3.IntegrityError:
                    st.error("El teléfono ya existe. Por favor, ingresa un número diferente.")

# --- Página: Ver Contactos & Exportar ---
elif page == "Ver Contactos & Exportar":
    st.title("Ver Contactos & Exportar")
    df_links = pd.read_sql_query("SELECT * FROM links_contactos", conn)
    if df_links.empty:
        st.warning("No existen links. Por favor, crea un Link Contactos primero.")
    else:
        df_links['display'] = df_links.apply(
            lambda row: f"{row['link_general']} - {row['marca']}", axis=1
        )
        link_selected = st.selectbox("Selecciona el Link Contactos", df_links['display'])
        selected_link = df_links[df_links['display'] == link_selected].iloc[0]
        link_id = selected_link["id"]
        
        st.markdown(f"**Fecha de Creación:** {selected_link['fecha_creacion']}")
        st.markdown(f"**Marca:** {selected_link['marca']}")
        st.markdown(f"**Descripción del Link:** {selected_link['descripcion']}")
        
        st.subheader("Filtros de Búsqueda")
        filter_nombre = st.text_input("Filtrar por Nombre")
        filter_auto = st.text_input("Filtrar por Auto")
        filter_telefono = st.text_input("Filtrar por Teléfono")
        
        query = "SELECT * FROM contactos WHERE id_link = ?"
        params = [link_id]
        if filter_nombre:
            query += " AND nombre LIKE ?"
            params.append(f"%{filter_nombre}%")
        if filter_auto:
            query += " AND auto LIKE ?"
            params.append(f"%{filter_auto}%")
        if filter_telefono:
            query += " AND telefono LIKE ?"
            params.append(f"%{filter_telefono}%")
        
        df_contactos = pd.read_sql_query(query, conn, params=params)
        
        st.subheader("Contactos Registrados")
        st.dataframe(df_contactos)
        
        if not df_contactos.empty:
            output = BytesIO()
            with pd.ExcelWriter(output, engine='xlsxwriter') as writer:
                df_contactos.to_excel(writer, index=False, sheet_name='Contactos')
            processed_data = output.getvalue()
            st.download_button(
                label="Descargar Excel",
                data=processed_data,
                file_name="contactos.xlsx",
                mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
            )


