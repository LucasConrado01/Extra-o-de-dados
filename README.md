# Extra-o-de-dados

!pip install xlsxwriter

import pandas as pd
import xlsxwriter
import zipfile
import io
import time
import numpy as np
import requests

### Primeira parte do código - Iremos encontrar as empresas que queremos trabalhar de um determinado setor, para isso, usamos as palavras chaves.

link = 'https://dados.cvm.gov.br/dados/CIA_ABERTA/CAD/DADOS/cad_cia_aberta.csv'

codigo_fonte = requests.get(link)

linhas = []

for i in codigo_fonte.text.split('\n'):
  linhas.append(i.strip().split(';'))

tabela = pd.DataFrame(linhas[1:], columns = linhas[0])

### Webscrapping da tabela gerada

searchfor = ['AZUL','AÉREO']   #Palavras-chaves para o filtro que será utilizado na tabela
empresa_buscada = tabela[tabela.DENOM_SOCIAL.str.contains('|'.join(searchfor), na = False)]

lista_cnpj = ['09.305.994/0001-29']
empresa_buscada_cnpj = tabela[tabela['CNPJ_CIA'].isin(lista_cnpj)]
empresas = list(empresa_buscada_cnpj['CD_CVM'])

### Segunda parte do código - Usada para pegar os resultados Tri/Anual das empresas selecionadas acima.

start_time = time.time()
a = 0
lista_de_listas = []

for j in empresas:

  lista_df = []

  demonstrativo = ['BPA','DRE','BPP','DFC_MI','DFC_MD']

  for k in demonstrativo:

    link = 'https://dados.cvm.gov.br/dados/CIA_ABERTA/DOC/DFP/DADOS/dfp_cia_aberta_2022.zip'
    arquivo_zip = requests.get(link)
    zf = zipfile.ZipFile(io.BytesIO(arquivo_zip.content))

    arquivo = 'dfp_cia_aberta_' + str(k) + '_con_2022.csv'

    dados = zf.open(arquivo)
    linhas_dre = dados.readlines()
    lines = [i.strip().decode('ISO-8859-1') for i in linhas_dre]
    lines = [i.split(';') for i in lines]
    df = pd.DataFrame(lines[1:], columns = lines[0])
    df['VC_AJUSTADO'] = pd.to_numeric(df['VL_CONTA'], errors = 'coerce')
    filtro = df[df['CD_CVM'] == str(j).zfill(6)]
    lista_df.append(filtro)

    print(f'Trabalhando com a empresa {j} no demonstrativo {k} e a dimensão do arquivo é {filtro.shape}')

  lista_de_listas.append(lista_df)

  #Passando as informações para o EXCEL
  writer = pd.ExcelWriter(f'Demonstrativo Empresa {str(j)}.xlsx', engine = 'xlsxwriter')
  lista_de_listas[a][0].to_excel(writer, sheet_name = 'BPA')
  lista_de_listas[a][1].to_excel(writer, sheet_name = 'DRE')
  lista_de_listas[a][2].to_excel(writer, sheet_name = 'BPP')
  lista_de_listas[a][3].to_excel(writer, sheet_name = 'DFC_MI')
  lista_de_listas[a][4].to_excel(writer, sheet_name = 'DFC_MD')
  a += 1

  print(f'O arquivo com as informações da empresa {str(j)} já foi exportado. \n')

  writer.close() #Fecha o arquivo

print ('O tempo para execução do código é de %s segundos ---' %(time.time() - start_time))
