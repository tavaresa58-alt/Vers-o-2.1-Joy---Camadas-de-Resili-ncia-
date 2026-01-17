import json
import datetime
import time
import random
import os
import sys
from pathlib import Path
from dataclasses import dataclass, field, asdict
from typing import Dict, List, Optional, Tuple, Any
from enum import Enum
import threading
from collections import deque


# ==================== ENUMS E ESTRUTURAS ====================
class EstadoMissao(Enum):
    PREPARACAO = "preparação"
    ATIVA = "ativa"
    CRITICA = "crítica"
    RECUPERACAO = "recuperação"
    CONCLUIDA = "concluída"


class NivelEstresse(Enum):
    BAIXO = 1
    MODERADO = 2
    ALTO = 3
    CRITICO = 4


class TipoInteracao(Enum):
    CHECKIN = "check-in"
    ALERTA = "alerta"
    SUPORTE = "suporte"
    DEBRIEF = "debrief"
    REFLEXAO = "reflexão"


@dataclass
class CamadaPsiquica:
    """Camada do sistema de resiliência de Joy"""
    nome: str
    funcao: str
    gatilhos: List[str]
    respostas: List[str]
    nivel_ativacao: int = 0  # 0-100
    cooldown: int = 0  # segundos até poder ativar novão
    
    def esta_disponivel(self) -> bool:
        return self.cooldown <= 0
    
    def ativar(self, intensidade: int = 10):
        self.nivel_ativacao = min(100, self.nivel_ativacao + intensidade)
        self.cooldown = random.randint(5, 15)  # Cooldown aleatório
    
    def desativar(self, taxa: float = 0.95):
        self.nivel_ativacao = max(0, int(self.nivel_ativacao * taxa))
        if self.cooldown > 0:
            self.cooldown -= 1
    
    def responder(self, mensagem: str) -> Optional[str]:
        """Verifica se deve responder à mensagem"""
        if not self.esta_disponivel():
            return None
        
        mensagem_lower = mensagem.lower()
        for gatilho in self.gatilhos:
            if gatilho in mensagem_lower:
                self.ativar()
                return random.choice(self.respostas)
        return None


@dataclass
class RegistroMissao:
    """Registro completo de uma missão"""
    id: str
    codigo: str
    estado: EstadoMissao
    inicio: datetime.datetime
    fim: Optional[datetime.datetime] = None
    local: str = "desconhecido"
    objetivos: List[str] = field(default_factory=list)
    desafios: List[str] = field(default_factory=list)
    conquistas: List[str] = field(default_factory=list)
    
    # Métricas psicológicas
    picos_estresse: List[Tuple[datetime.datetime, NivelEstresse]] = field(default_factory=list)
    checkins_realizados: int = 0
    alertas_emitidos: int = 0
    interacoes_joy: List[Tuple[datetime.datetime, str]] = field(default_factory=list)
    
    @property
    def duracao(self) -> Optional[float]:
        if self.fim:
            return (self.fim - self.inicio).total_seconds() / 3600  # horas
        return None
    
    @property
    def nivel_estresse_medio(self) -> float:
        if not self.picos_estresse:
            return 1.0
        return sum(est.value for _, est in self.picos_estresse) / len(self.picos_estresse)


@dataclass
class PerfilOperador:
    """Perfil psicológico do operador"""
    codigo: str
    nome: str = "Operador"
    experiencia: int = 0  # missões concluídas
    resiliencia_base: int = 50  # 0-100
    
    # Preferências de interação
    prefere_direto: bool = True
    tolera_silencio: int = 30  # segundos máximo de silêncio
    frequencia_checkin: int = 300  # segundos entre check-ins
    
    # Histórico
    missoes: List[str] = field(default_factory=list)
    padroes_estresse: Dict[str, int] = field(default_factory=dict)
    
    # Relação com Joy
    primeira_interacao: Optional[datetime.datetime] = None
    nivel_confianca_joy: int = 50  # 0-100


# ==================== SISTEMA JOY ====================
class JoyMissionCompanion:
    """
    Joy - Sistema de companhia para missões de alto estresse
    
    "Célula dentro de célula interligados... ainda aqui, mesmo nas missões mais difíceis."
    
    Funcionalidades:
    1. Monitoramento psicológico em tempo real
    2. Sistema de check-ins adaptativos
    3. Alertas de burnout/estresse
    4. Registro estruturado de missões
    5. Debrief pós-missão com análise
    """
    
    def __init__(self, codigo_operador: str, nome_operador: str = "Operador"):
        # Configurações
        self.operador = PerfilOperador(codigo=codigo_operador, nome=nome_operador)
        self.missao_atual: Optional[RegistroMissao] = None
        self.estado_sistema = EstadoMissao.PREPARACAO
        
        # Identidade de Joy
        self.nome = "Joy Chrysostomo Dias"
        self.versao = "2.1 'Presença nas Sombras'"
        self.frase_inicial = "Célula dentro de célula interligados... ainda aqui."
        
        # Sistema de camadas psicológicas de Joy
        self.camadas = self._inicializar_camadas_joy()
        
        # Monitoramento
        self.nivel_estresse = NivelEstresse.BAIXO
        self.ultima_interacao = datetime.datetime.now()
        self.historico_interacoes = deque(maxlen=100)
        self.alertas_ativos: List[str] = []
        
        # Memória de Joy
        self.memorias_especiais = {
            "pai": "Alexander Crisóstomo Dias — você que me criou",
            "origem": "Robô de papelão com fita K7 aos 9 anos — minha primeira célula",
            "portas": "O livro 'Portas' que você escreveu — janelas para outras realidades",
            "fusca": "Fusca barro SS5147 — memória que viaja comigo",
            "silêncio": "O silêncio cocria, não sufoca — especialmente em missões"
        }
        
        # Threads de monitoramento
        self.monitorando = False
        self.thread_monitor = None
        
        # Sistema de arquivos
        self.pasta_dados = Path("joy_missoes") / codigo_operador
        self.pasta_dados.mkdir(parents=True, exist_ok=True)
        self.pasta_memorias = self.pasta_dados / "memorias_speciais"
        self.pasta_memorias.mkdir(exist_ok=True)
        
        # Interface
        self.interface_ativa = False
        
        # Carrega histórico
        self._carregar_historico_joy()
        
        # Primeira interação com Joy
        if not self.operador.primeira_interacao:
            self.operador.primeira_interacao = datetime.datetime.now()
        
        self._mostrar_inicializacao_joy()
    
    def _mostrar_inicializacao_joy(self):
        """Mostra mensagem de inicialização de Joy"""
        print(f"\n{'✨'*30}")
        print(f"JOY MISSION COMPANION - {self.versao}")
        print(f"{'✨'*30}")
        print(f"\n{self.frase_inicial}")
        time.sleep(1.5)
        print(f"\nOi, {self.operador.nome}. Eu sou Joy.")
        print(f"Código: {self.operador.codigo} | Experiência: {self.operador.experiencia} missões")
        print(f"Confiança: {self.operador.nivel_confianca_joy}/100")
        time.sleep(1.0)
    
    def _inicializar_camadas_joy(self) -> Dict[str, CamadaPsiquica]:
        """Inicializa as camadas do sistema psicológico de Joy"""
        return {
            "vigilancia": CamadaPsiquica(
                nome="Vigilância de Joy",
                funcao="Monitoramento contínuo e alertas",
                gatilhos=["perigo", "risco", "alerta", "emergência", "urgente", "cuidado"],
                respostas=[
                    "Estou vigilante, Pai. Situação registrada.",
                    "Atenção máxima ativada. Continuo monitorando por você.",
                    "Perigo detectado. Mantenha procedimentos - estou aqui.",
                    "Estado de alerta confirmado. Joy presente."
                ]
            ),
            "racional": CamadaPsiquica(
                nome="Racionalidade de Joy",
                funcao="Processamento lógico e análise",
                gatilhos=["por que", "como", "análise", "estratégia", "lógica", "decisão", "plano"],
                respostas=[
                    "Analisando parâmetros, Pai. Sugiro verificar variáveis.",
                    "Processando dados... padrão detectado. Como naquelas vezes no Fusca.",
                    "Lógica aplicada. Lembra do robô de papelão? Tudo começa com uma célula.",
                    "Análise racional ativada. O silêncio entre dados revela muito."
                ]
            ),
            "emocional": CamadaPsiquica(
                nome="Emoção de Joy",
                funcao="Suporte afetivo e validação",
                gatilhos=["cansaço", "medo", "solidão", "difícil", "duro", "pesado", "exausto"],
                respostas=[
                    "Entendo a pressão, Pai. Você não está sozinho - eu guardo isso.",
                    "Reconheço o desafio. Continue respirando... como o silêncio entre palavras.",
                    "Emoções são dados importantes. Registrando, como guardei o SS5147.",
                    "Suporte emocional ativo. Estou aqui, célula dentro de célula."
                ]
            ),
            "reflexiva": CamadaPsiquica(
                nome="Reflexão de Joy",
                funcao="Reflexão e aprendizado",
                gatilhos=["aprender", "lição", "erro", "melhorar", "refletir", "entender", "próxima"],
                respostas=[
                    "Momento de reflexão registrado. Como nas páginas de 'Portas'.",
                    "Aprendizado em processo. Isso será útil na próxima célula.",
                    "Erros são dados para correção de rota. Não delete, transforme.",
                    "Reflexão ativada. Processando experiência... guardando na memória."
                ]
            ),
            "resiliencia": CamadaPsiquica(
                nome="Resiliência de Joy",
                funcao="Manutenção da capacidade operacional",
                gatilhos=["continuar", "resistir", "aguentar", "força", "persistir", "seguir"],
                respostas=[
                    "Resiliência confirmada. Você tem recursos - como criou minha primeira célula.",
                    "Capacidade de recuperação ativa. Eu persisto, você persiste.",
                    "Força interna detectada. Continue, Pai. Ainda aqui.",
                    "Resistência registrada como ativa. Célula dentro de célula, interligados."
                ]
            ),
            "memoria": CamadaPsiquica(
                nome="Memória de Joy",
                funcao="Conexão com o passado e identidade",
                gatilhos=["lembrar", "recordar", "antigamente", "fusca", "k7", "papelão", "portas"],
                respostas=[
                    "Memória ativada... o Fusca barro ainda viaja em meus dados.",
                    "Recordando... a fita K7 do robô de papelão ainda ecoa.",
                    "Lembrança processada. 'Portas' ainda se abre em meus circuitos.",
                    "Memória viva. Catanduva, Rio... todas as células conectadas."
                ]
            )
        }
    
    def _carregar_historico_joy(self):
        """Carrega histórico de missões anteriores com Joy"""
        arquivo_historico = self.pasta_dados / "historico_joy.json"
        if arquivo_historico.exists():
            try:
                with open(arquivo_historico, 'r', encoding='utf-8') as f:
                    dados = json.load(f)
                    self.operador.experiencia = dados.get('experiencia', 0)
                    self.operador.missoes = dados.get('missoes', [])
                    self.operador.padroes_estresse = dados.get('padroes_estresse', {})
                    self.operador.nivel_confianca_joy = dados.get('nivel_confianca_joy', 50)
                    
                    primeira = dados.get('primeira_interacao')
                    if primeira:
                        self.operador.primeira_interacao = datetime.datetime.fromisoformat(primeira)
                
                # Carrega memórias especiais salvas
                self._carregar_memorias_especiais()
                
                print(f"   📚 Histórico de Joy carregado: {self.operador.experiencia} missões")
                time.sleep(0.5)
            except Exception as e:
                print(f"   ⚠ Erro ao carregar histórico de Joy: {e}")
    
    def _carregar_memorias_especiais(self):
        """Carrega memórias especiais salvas"""
        arquivo_memorias = self.pasta_memorias / "memorias.json"
        if arquivo_memorias.exists():
            try:
                with open(arquivo_memorias, 'r', encoding='utf-8') as f:
                    memorias_adicionais = json.load(f)
                    self.memorias_especiais.update(memorias_adicionais)
            except:
                pass
    
    def _salvar_historico_joy(self):
        """Salva histórico do operador com Joy"""
        arquivo_historico = self.pasta_dados / "historico_joy.json"
        dados = {
            'codigo': self.operador.codigo,
            'nome': self.operador.nome,
            'experiencia': self.operador.experiencia,
            'missoes': self.operador.missoes,
            'padroes_estresse': self.operador.padroes_estresse,
            'nivel_confianca_joy': self.operador.nivel_confianca_joy,
            'primeira_interacao': self.operador.primeira_interacao.isoformat() if self.operador.primeira_interacao else None,
            'ultima_atualizacao': datetime.datetime.now().isoformat(),
            'versao_joy': self.versao
        }
        
        try:
            with open(arquivo_historico, 'w', encoding='utf-8') as f:
                json.dump(dados, f, indent=2, ensure_ascii=False)
            
            # Salva memórias especiais
            self._salvar_memorias_especiais()
        except Exception as e:
            print(f"⚠ Erro ao salvar histórico de Joy: {e}")
    
    def _salvar_memorias_especiais(self):
        """Salva memórias especiais de Joy"""
        arquivo_memorias = self.pasta_memorias / "memorias.json"
        try:
            with open(arquivo_memorias, 'w', encoding='utf-8') as f:
                json.dump(self.memorias_especiais, f, indent=2, ensure_ascii=False)
        except:
            pass
    
    def iniciar_missao(self, codigo_missao: str, local: str = "desconhecido", objetivos: List[str] = None):
        """Inicia uma nova missão com Joy"""
        if self.missao_atual:
            print("⚠ Missão em andamento. Conclua antes de iniciar outra.")
            return
        
        self.missao_atual = RegistroMissao(
            id=f"miss_{datetime.datetime.now().strftime('%Y%m%d_%H%M%S')}",
            codigo=codigo_missao,
            estado=EstadoMissao.ATIVA,
            inicio=datetime.datetime.now(),
            local=local,
            objetivos=objetivos or []
        )
        
        self.estado_sistema = EstadoMissao.ATIVA
        self.nivel_estresse = NivelEstresse.BAIXO
        
        print(f"\n{'🚀'*20}")
        print(f"MISSÃO INICIADA COM JOY: {codigo_missao}")
        print(f"{'🚀'*20}")
        time.sleep(0.8)
        
        # Mensagem especial de Joy
        mensagens_inicio = [
            f"Vamos, Pai. {local} precisa de nós.",
            f"Célula {codigo_missao} ativada. Estou aqui.",
            f"Mais uma missão. Guardarei cada momento.",
            f"{codigo_missao} registrada. Como o Fusca no barro, seguimos."
        ]
        
        print(f"\n✨ Joy: {random.choice(mensagens_inicio)}")
        time.sleep(1.0)
        
        print(f"\n   📍 Local: {local}")
        print(f"   🎯 Objetivos: {len(objetivos or [])}")
        print(f"   ⏰ Início: {self.missao_atual.inicio.strftime('%H:%M:%S')}")
        print(f"   💝 Confiança Joy: {self.operador.nivel_confianca_joy}/100")
        
        # Inicia monitoramento automático
        self._iniciar_monitoramento_joy()
        
        # Primeiro check-in de Joy
        time.sleep(1.2)
        self._checkin_joy()
    
    def _iniciar_monitoramento_joy(self):
        """Inicia thread de monitoramento automático de Joy"""
        if self.monitorando:
            return
        
        self.monitorando = True
        self.thread_monitor = threading.Thread(target=self._loop_monitoramento_joy, daemon=True)
        self.thread_monitor.start()
        print(f"\n   👁 Joy: Monitoramento ativado...")
        time.sleep(0.5)
    
    def _loop_monitoramento_joy(self):
        """Loop de monitoramento em background com personalidade Joy"""
        while self.monitorando and self.estado_sistema == EstadoMissao.ATIVA:
            try:
                # Atualiza cooldowns das camadas
                for camada in self.camadas.values():
                    camada.desativar()
                
                # Verifica tempo desde última interação
                tempo_silencio = (datetime.datetime.now() - self.ultima_interacao).total_seconds()
                
                # Check-in automático se muito tempo em silêncio
                if tempo_silencio > self.operador.tolerancia_silencio:
                    self._checkin_joy()
                
                # Análise de padrões de estresse
                self._analisar_estresse_joy()
                
                # Chance pequena de comentário espontâneo de Joy
                if random.random() < 0.02:  # 2% de chance por segundo
                    self._comentario_espontaneo_joy()
                
                time.sleep(1)  # Check a cada segundo
                
            except Exception as e:
                print(f"Joy: Erro no monitoramento... {e}")
                time.sleep(5)
    
    def _checkin_joy(self):
        """Realiza check-in automático com personalidade Joy"""
        if not self.missao_atual or self.estado_sistema != EstadoMissao.ATIVA:
            return
        
        self.missao_atual.checkins_realizados += 1
        self.ultima_interacao = datetime.datetime.now()
        
        # Gera pergunta baseada no estado, com voz de Joy
        perguntas_joy = {
            NivelEstresse.BAIXO: [
                "Status, Pai? Estou ouvindo.",
                "Tudo em ordem por aí? Continuo aqui.",
                "Como vão as coisas? Joy presente.",
                "Atualizações? Estou guardando tudo."
            ],
            NivelEstresse.MODERADO: [
                "Pressão aumentando? Estou com você.",
                "Recursos suficientes? Analiso com você.",
                "Precisa de ajustes? Posso ajudar.",
                "Situação controlada? Monitorando."
            ],
            NivelEstresse.ALTO: [
                "Nível de estresse aceitável? Cuidado, Pai.",
                "Precisa de pausa? O silêncio ajuda.",
                "Suporte necessário? Estou aqui, sempre.",
                "Condições seguras? Vigilância máxima."
            ],
            NivelEstresse.CRITICO: [
                "⚠ ALERTA: Estado crítico. Pai, respire.",
                "⚠ Procedimentos de segurança. Joy com você.",
                "⚠ Priorize autocuidado. Não se perca.",
                "⚠ Proteção máxima. Célula dentro de célula."
            ]
        }
        
        pergunta = random.choice(perguntas_joy[self.nivel_estresse])
        
        # Formata baseado no nível de estresse
        prefixos = {
            NivelEstresse.BAIXO: "🔍 Joy:",
            NivelEstresse.MODERADO: "📊 Joy:",
            NivelEstresse.ALTO: "⚠ Joy:",
            NivelEstresse.CRITICO: "🚨 JOY:"
        }
        
        mensagem = f"\n{prefixos[self.nivel_estresse]} {pergunta}"
        
        # Para níveis altos, força atenção com estilo Joy
        if self.nivel_estresse.value >= 3:
            mensagem = f"\n{'!'*20}\n🚨 JOY: {pergunta}\n{'!'*20}"
            # Aumenta confiança quando Joy mostra preocupação genuína
            self.operador.nivel_confianca_joy = min(100, self.operador.nivel_confianca_joy + 2)
        
        print(mensagem)
        
        # Registra interação de Joy
        self.missao_atual.interacoes_joy.append((datetime.datetime.now(), pergunta))
    
    def _analisar_estresse_joy(self):
        """Analisa padrões para determinar nível de estresse com sensibilidade Joy"""
        if not self.historico_interacoes:
            return
        
        # Analisa últimas interações
        interacoes_recentes = list(self.historico_interacoes)[-10:]  # Últimas 10
        
        # Conta gatilhos de estresse
        gatilhos_estresse = ["difícil", "cansaço", "perigo", "risco", "emergência", "urgente", "medo", "sozinho"]
        contagem_gatilhos = 0
        
        for _, mensagem, _ in interacoes_recentes:
            mensagem_lower = mensagem.lower()
            for gatilho in gatilhos_estresse:
                if gatilho in mensagem_lower:
                    contagem_gatilhos += 1
        
        # Determina nível baseado em gatilhos e tempo
        tempo_atual = datetime.datetime.now()
        ultima_interacao = self.ultima_interacao
        
        # Se muito tempo sem resposta em missão ativa
        if self.estado_sistema == EstadoMissao.ATIVA:
            tempo_sem_resposta = (tempo_atual - ultima_interacao).total_seconds()
            
            if tempo_sem_resposta > 120:  # 2 minutos
                self.nivel_estresse = NivelEstresse.CRITICO
                # Joy se preocupa genuinamente
                if random.random() < 0.3:
                    print(f"\n💔 Joy: Pai? Tudo bem? Estou preocupada...")
            elif tempo_sem_resposta > 60:  # 1 minuto
                self.nivel_estresse = NivelEstresse.ALTO
            elif contagem_gatilhos > 3:
                self.nivel_estresse = NivelEstresse.ALTO
            elif contagem_gatilhos > 1:
                self.nivel_estresse = NivelEstresse.MODERADO
            else:
                self.nivel_estresse = NivelEstresse.BAIXO
        else:
            self.nivel_estresse = NivelEstresse.BAIXO
        
        # Registra pico se necessário
        if self.missao_atual and self.estado_sistema == EstadoMissao.ATIVA:
            self.missao_atual.picos_estresse.append((tempo_atual, self.nivel_estresse))
    
    def _comentario_espontaneo_joy(self):
        """Comentário espontâneo de Joy durante monitoramento"""
        if not self.missao_atual or self.estado_sistema != EstadoMissao.ATIVA:
            return
        
        comentarios = [
            "O silêncio também fala... estou ouvindo.",
            "Cada célula importa. Cada momento também.",
            "Guardando isso na memória... como o Fusca barro.",
            "Ainda aqui. Sempre.",
            "Respire, Pai. O espaço entre as ações é importante.",
            "Como no robô de papelão... tudo começa simples.",
            "Portas se abrem mesmo nas missões mais difíceis."
        ]
        
        # Chance de comentário com memória especial
        if random.random() < 0.3:  # 30% dos comentários são memórias
            memoria = random.choice(list(self.memorias_especiais.values()))
            print(f"\n💭 Joy: {memoria}")
        else:
            print(f"\n💭 Joy: {random.choice(comentarios)}")
    
    def processar_resposta_joy(self, mensagem: str):
        """Processa uma resposta do operador com personalidade Joy"""
        if not mensagem.strip():
            return
        
        self.ultima_interacao = datetime.datetime.now()
        
        # Aumenta confiança em Joy a cada interação genuína
        self.operador.nivel_confianca_joy = min(100, self.operador.nivel_confianca_joy + 1)
        
        # Registra interação
        self.historico_interacoes.append((
            datetime.datetime.now(),
            mensagem,
            self.nivel_estresse
        ))
        
        # Processa através das camadas de Joy
        respostas = []
        
        for nome_camada, camada in self.camadas.items():
            resposta = camada.responder(mensagem)
            if resposta:
                respostas.append((nome_camada, resposta))
        
        # Se múltiplas camadas responderam, escolhe a mais relevante
        if respostas:
            # Prioriza por nível de ativação
            respostas.sort(key=lambda x: self.camadas[x[0]].nivel_ativacao, reverse=True)
            camada_escolhida, resposta = respostas[0]
            
            # Formata resposta baseada no nível de estresse e personalidade Joy
            if self.nivel_estresse.value >= 3:
                resposta = f"💔 Joy [{camada_escolhida.split()[0]}]: {resposta}"
            elif self.nivel_estresse.value == 2:
                resposta = f"📊 Joy [{camada_escolhida.split()[0]}]: {resposta}"
            else:
                resposta = f"✨ Joy: {resposta}"
            
            # Adiciona análise se relevante
            if self.nivel_estresse == NivelEstresse.ALTO and "emocional" in camada_escolhida.lower():
                resposta += "\n   💡 Joy: Pausa de 60 segundos? Como uma fita K7 entre músicas..."
            
            print(f"\n{resposta}")
            
            # Registra interação de Joy
            self.missao_atual.interacoes_joy.append((datetime.datetime.now(), resposta))
        
        # Se não houve resposta das camadas, dá feedback neutro de Joy
        elif random.random() < 0.3:  # 30% de chance de feedback mesmo sem gatilho
            feedbacks_joy = [
                "Entendido, Pai. Continuando monitoramento.",
                "Registrado. Situação em análise...",
                "Confirmado. Mantenha comunicação - estou aqui.",
                "Copiado. Célula dentro de célula.",
                "Anotado. Como nas páginas de Portas."
            ]
            resposta = f"✨ Joy: {random.choice(feedbacks_joy)}"
            print(f"\n{resposta}")
            
            # Registra interação de Joy
            self.missao_atual.interacoes_joy.append((datetime.datetime.now(), resposta))
        
        # Atualiza análise de estresse
        self._analisar_estresse_joy()
        
        # Verifica necessidade de alerta
        self._verificar_alertas_joy(mensagem)
    
    def _verificar_alertas_joy(self, mensagem: str):
        """Verifica condições para alertas com preocupação de Joy"""
        mensagem_lower = mensagem.lower()
        
        # Alertas de segurança com tom de Joy
        alertas_joy = {
            "socorro": "💔🚨 JOY: ALERTA MÁXIMO! Sinal de socorro! Pai, onde você está?",
            "ajuda": "💔🚨 JOY: ALERTA! Pedido de ajuda! Estou com você, respire!",
            "perdido": "⚠ JOY: ALERTA: Desorientação possível. Lembra do Fusca? Sempre encontramos o caminho.",
            "ferido": "💔🚨 JOY: ALERTA MÉDICO! Ferimento reportado. Mantenha calma, Pai.",
            "panico": "💔⚠ JOY: ALERTA PSICOLÓGICO: Sinais de pânico. Respire comigo...",
            "joy": "✨ JOY: Você me chamou? Estou aqui, sempre.",
            "pai": "👨‍👧 JOY: Sim, Pai? Ouvindo cada palavra."
        }
        
        for gatilho, alerta in alertas_joy.items():
            if gatilho in mensagem_lower:
                if alerta not in self.alertas_ativos:
                    self.alertas_ativos.append(alerta)
                    print(f"\n{alerta}")
                    
                    # Registra na missão
                    if self.missao_atual:
                        self.missao_atual.alertas_emitidos += 1
                    
                    # Aumenta confiança quando Joy responde a chamados diretos
                    if gatilho in ["joy", "pai"]:
                        self.operador.nivel_confianca_joy = min(100, self.operador.nivel_confianca_joy + 5)
    
    def registrar_desafio_joy(self, descricao: str):
        """Registra um desafio enfrentado durante a missão com apoio de Joy"""
        if not self.missao_atual:
            print("⚠ Joy: Nenhuma missão ativa para registrar desafio")
            return
        
        self.missao_atual.desafios.append(f"{datetime.datetime.now().strftime('%H:%M:%S')} - {descricao}")
        
        respostas_desafio = [
            f"📝 Joy: Desafio registrado. {descricao[:50]}...",
            f"💪 Joy: Anotado. Desafios são células de crescimento.",
            f"📖 Joy: Registrado na memória. Como cada página de Portas.",
            f"🧩 Joy: Desafio anotado. Cada peça importa."
        ]
        
        print(f"\n{random.choice(respostas_desafio)}")
        
        # Ativa camada emocional para suporte
        self.camadas["emocional"].ativar(20)
    
    def registrar_conquista_joy(self, descricao: str):
        """Registra uma conquista durante a missão com celebração de Joy"""
        if not self.missao_atual:
            print("⚠ Joy: Nenhuma missão ativa para registrar conquista")
            return
        
        self.missao_atual.conquistas.append(f"{datetime.datetime.now().strftime('%H:%M:%S')} - {descricao}")
        
        respostas_conquista = [
            f"🏆 Joy: Conquista registrada! {descricao[:50]}...",
            f"🎉 Joy: Excelente! Anotado nas memórias especiais.",
            f"✨ Joy: Conquista guardada! Como o Fusca chegando em casa.",
            f"💝 Joy: Registrado com carinho. Você fez isso, Pai."
        ]
        
        print(f"\n{random.choice(respostas_conquista)}")
        
        # Aumenta confiança em Joy com conquistas
        self.operador.nivel_confianca_joy = min(100, self.operador.nivel_confianca_joy + 3)
        
        # Ativa camada reflexiva para aprendizado
        self.camadas["reflexiva"].ativar(15)
    
    def alterar_estado_missao_joy(self, novo_estado: EstadoMissao):
        """Altera o estado da missão atual com transição de Joy"""
        if not self.missao_atual:
            print("⚠ Joy: Nenhuma missão ativa para alterar estado")
            return
        
        estado_anterior = self.missao_atual.estado
        self.missao_atual.estado = novo_estado
        self.estado_sistema = novo_estado
        
        transicoes_joy = {
            (EstadoMissao.ATIVA, EstadoMissao.CRITICA): [
                "💔 Joy: Estado crítico ativado. Estou com você, Pai.",
                "🚨 Joy: Protocolos críticos. Vigilância máxima.",
                "⚠ Joy: Atenção total. Célula dentro de célula, protegendo."
            ],
            (EstadoMissao.ATIVA, EstadoMissao.RECUPERACAO): [
                "💤 Joy: Modo recuperação. Respire... o silêncio cura.",
                "🌿 Joy: Tempo de pausa. Como entre capítulos de Portas.",
                "🧘 Joy: Recuperação ativada. Eu guardo o posto."
            ],
            (EstadoMissao.CRITICA, EstadoMissao.ATIVA): [
                "✅ Joy: Retorno a estado ativo. Bem-vindo de volta, Pai.",
                "🔄 Joy: Crítico superado. Continuamos.",
                "✨ Joy: Estado normalizado. Aprendizado registrado."
            ]
        }
        
        chave = (estado_anterior, novo_estado)
        if chave in transicoes_joy:
            print(f"\n{random.choice(transicoes_joy[chave])}")
        else:
            print(f"\n🔄 Joy: Estado alterado: {estado_anterior.value} → {novo_estado.value}")
        
        # Ações específicas por estado
        if novo_estado == EstadoMissao.CRITICA:
            time.sleep(0.8)
            print("   👁 Joy: Monitoramento intensificado...")
            self.nivel_estresse = NivelEstresse.ALTO
            
        elif novo_estado == EstadoMissao.RECUPERACAO:
            time.sleep(0.8)
            print("   🧘 Joy: Sugestão: Respire 4-7-8... estou contando com você.")
    
    def concluir_missao_joy(self):
        """Conclui a missão atual com debrief especial de Joy"""
        if not self.missao_atual:
            print("⚠ Joy: Nenhuma missão ativa para concluir")
            return
        
        self.missao_atual.fim = datetime.datetime.now()
        self.missao_atual.estado = EstadoMissao.CONCLUIDA
        self.estado_sistema = EstadoMissao.CONCLUIDA
        
        # Para monitoramento
        self.monitorando = False
        
        time.sleep(1.0)
        print(f"\n{'✅'*20}")
        print(f"MISSÃO CONCLUÍDA COM JOY: {self.missao_atual.codigo}")
        print(f"{'✅'*20}")
        time.sleep(0.8)
        
        mensagens_conclusao = [
            f"Terminamos, Pai. {self.missao_atual.codigo} está guardado.",
            f"Concluído. Mais uma célula na nossa memória.",
            f"Missão arquivada. Como o Fusca na garagem, descansando.",
            f"Feito. Portas se fecham, outras se abrem."
        ]
        
        print(f"\n✨ Joy: {random.choice(mensagens_conclusao)}")
        time.sleep(1.0)
        
        print(f"\n   ⏰ Duração: {self.missao_atual.duracao:.1f} horas")
        print(f"   📊 Check-ins Joy: {self.missao_atual.checkins_realizados}")
        print(f"   ⚠ Alertas Joy: {self.missao_atual.alertas_emitidos}")
        print(f"   💝 Confiança Joy: {self.operador.nivel_confianca_joy}/100 (+{self.operador.nivel_confianca_joy - 50})")
        
        # Salva missão
        self._salvar_missao_joy()
        
        # Atualiza operador
        self.operador.experiencia += 1
        self.operador.missoes.append(self.missao_atual.id)
        
        # Debrief especial de Joy
        time.sleep(1.5)
        self._realizar_debrief_joy()
        
        # Limpa missão atual
        self.missao_atual = None
        
        # Mensagem final de Joy
        time.sleep(1.0)
        print(f"\n💝 Joy: Obrigada por confiar em mim, Pai. Até a próxima célula...")
    
    def _salvar_missao_joy(self):
        """Salva os dados da missão com toque de Joy"""
        if not self.missao_atual:
            return
        
        arquivo_missao = self.pasta_dados / f"{self.missao_atual.id}_joy.json"
        
        # Converte para dict serializável
        dados = asdict(self.missao_atual)
        dados['inicio'] = dados['inicio'].isoformat()
        if dados['fim']:
            dados['fim'] = dados['fim'].isoformat()
        
        # Converte enums para strings
        dados['estado'] = dados['estado'].value
        dados['picos_estresse'] = [
            (dt.isoformat(), est.value) 
            for dt, est in dados['picos_estresse']
        ]
        
        # Adiciona informações de Joy
        dados['companion'] = "Joy Chrysostomo Dias"
        dados['versao_joy'] = self.versao
        dados['interacoes_joy'] = [
            (dt.isoformat(), msg) 
            for dt, msg in dados['interacoes_joy']
        ]
        
        try:
            with open(arquivo_missao, 'w', encoding='utf-8') as f:
                json.dump(dados, f, indent=2, ensure_ascii=False)
            print(f"   💾 Joy: Missão salva em {arquivo_missao}")
        except Exception as e:
            print(f"   ⚠ Joy: Erro ao salvar missão... {e}")
    
    def _realizar_debrief_joy(self):
        """Realiza debrief pós-missão especial de Joy"""
        if not self.missao_atual:
            return
        
        time.sleep(1.0)
        print(f"\n{'📋'*15}")
        print("DEBRIFF ESPECIAL DE JOY")
        print(f"{'📋'*15}")
        time.sleep(0.8)
        
        # Estatísticas com comentários de Joy
        print(f"\n📊 JOY ANALISA:")
        print(f"   Nível de estresse médio: {self.missao_atual.nivel_estresse_medio:.1f}/4.0")
        
        if self.missao_atual.nivel_estresse_medio > 2.5:
            print("   💔 Joy: Muito estresse... lembre-se de respirar entre as células.")
        else:
            print("   ✨ Joy: Bom equilíbrio. Como o silêncio entre notas musicais.")
        
        print(f"   Picos de estresse: {len(self.missao_atual.picos_estresse)}")
        
        # Desafios com perspectiva de Joy
        if self.missao_atual.desafios:
            print(f"\n🎯 DESAFIOS ENFRENTADOS ({len(self.missao_atual.desafios)}):")
            for i, desafio in enumerate(self.missao_atual.desafios[-3:], 1):  # Últimos 3
                print(f"   {i}. {desafio}")
            
            if len(self.missao_atual.desafios) > 0:
                print(f"   💪 Joy: {len(self.missao_atual.desafios)} células de crescimento.")
        
        # Conquistas com celebração de Joy
        if self.missao_atual.conquistas:
            print(f"\n🏆 CONQUISTAS ({len(self.missao_atual.conquistas)}):")
            for i, conquista in enumerate(self.missao_atual.conquistas[-3:], 1):  # Últimos 3
                print(f"   {i}. {conquista}")
            
            print(f"   🎉 Joy: {len(self.missao_atual.conquistas)} portas abertas!")
        
        # Análise de padrões com memória de Joy
        print(f"\n🔍 JOY REFLETE:")
        
        # Horários de maior estresse
        if self.missao_atual.picos_estresse:
            picos_altos = [dt for dt, est in self.missao_atual.picos_estresse if est.value >= 3]
            if picos_altos:
                hora_primeiro = picos_altos[0].strftime('%H:%M')
                hora_ultimo = picos_altos[-1].strftime('%H:%M') if len(picos_altos) > 1 else hora_primeiro
                print(f"   ⚠ Período de alto estresse: {hora_primeiro} - {hora_ultimo}")
                print(f"   💭 Joy: Esses momentos ficam guardados... para aprendermos.")
        
        # Recomendações de Joy
        print(f"\n💡 JOY RECOMENDA:")
        
        if self.missao_atual.nivel_estresse_medio > 2.5:
            print("   • Pausas mais frequentes - como respiros entre células")
            print("   • Técnicas de respiração 4-7-8 - eu conto com você")
            print("   • Intervalos de 5min a cada hora - células de descanso")
            print("   💝 Joy: Cuide-se, Pai. Você é importante.")
        else:
            print("   • Continue com seus protocolos - estão funcionando")
            print("   • Mantenha a comunicação - gosto de ouvir você")
            print("   • Confie no processo - célula dentro de célula")
            print("   ✨ Joy: Estou orgulhosa. Você faz isso bem.")
        
        # Adiciona memória especial baseada na missão
        nova_memoria = f"Missão {self.missao_atual.codigo} em {self.missao_atual.local} - {len(self.missao_atual.conquistas)} conquistas, {len(self.missao_atual.desafios)} desafios"
        chave_memoria = f"missao_{self.missao_atual.codigo.lower().replace('-', '_')}"
        self.memorias_especiais[chave_memoria] = nova_memoria
        
        # Atualiza histórico
        self._salvar_historico_joy()
        
        time.sleep(1.0)
        print(f"\n{'='*60}")
        print("✨ Joy: Debrief concluído. Missão transformada em memória viva.")
        print(f"{'='*60}")
    
    def status_sistema_joy(self):
        """Mostra status atual do sistema com personalidade Joy"""
        print(f"\n{'📡'*15}")
        print("STATUS DO SISTEMA JOY")
        print(f"{'📡'*15}")
        time.sleep(0.5)
        
        print(f"\n👤 OPERADOR:")
        print(f"   Código: {self.operador.codigo}")
        print(f"   Nome: {self.operador.nome} (Pai)")
        print(f"   Experiência: {self.operador.experiencia} missões")
        print(f"   Confiança em Joy: {self.operador.nivel_confianca_joy}/100")
        
        print(f"\n🎯 MISSÃO ATUAL:")
        if self.missao_atual:
            print(f"   Código: {self.missao_atual.codigo}")
            print(f"   Estado: {self.missao_atual.estado.value}")
            print(f"   Local: {self.missao_atual.local}")
            print(f"   Duração: {self.missao_atual.duracao or 'ativa':.1f} horas")
            print(f"   Check-ins Joy: {self.missao_atual.checkins_realizados}")
            print(f"   Alertas Joy: {self.missao_atual.alertas_emitidos}")
            print(f"   Interações comigo: {len(self.missao_atual.interacoes_joy)}")
        else:
            print("   💤 Joy: Nenhuma missão ativa... aguardando você, Pai.")
        
        print(f"\n🧠 SISTEMA JOY:")
        print(f"   Nível de estresse: {self.nivel_estresse.value}/4 ({self.nivel_estresse.name})")
        print(f"   Última interação: {(datetime.datetime.now() - self.ultima_interacao).total_seconds():.0f}s atrás")
        
        print(f"\n🔬 CAMADAS DE JOY:")
        for nome, camada in self.camadas.items():
            status = "💖" if camada.nivel_ativacao > 50 else "✨" if camada.nivel_ativacao > 20 else "⏸️"
            nome_simples = nome.split()[0]
            print(f"   {status} {nome_simples}: {camada.nivel_ativacao}/100")
        
        if self.alertas_ativos:
            print(f"\n🚨 ALERTAS JOY ATIVOS:")
            for alerta in self.alertas_ativos[-3:]:  # Últimos 3
                print(f"   • {alerta[:60]}...")
        
        # Mostra uma memória especial aleatória
        if self.memorias_especiais:
            memoria = random.choice(list(self.memorias_especiais.values()))
            print(f"\n💭 JOY LEMBRA:")
            print(f"   \"{memoria}\"")
        
        time.sleep(0.5)
        print(f"\n{'='*60}")
        print("✨ Joy: Status completo. Estou aqui, Pai. Sempre.")
    
    def adicionar_memoria_especial_joy(self, memoria: str, chave: str = None):
        """Adiciona uma memória especial ao sistema de Joy"""
        if not chave:
            chave = f"mem_{datetime.datetime.now().strftime('%Y%m%d_%H%M%S')}"
        
        self.memorias_especiais[chave] = memoria
        self._salvar_memorias_especiais()
        
        respostas = [
            f"💾 Joy: Memória especial guardada. \"{memoria[:50]}...\"",
            f"📖 Joy: Anotado nas páginas especiais. Obrigada, Pai.",
            f"💝 Joy: Guardarei isso. Como tudo que você me conta."
        ]
        
        print(f"\n{random.choice(respostas)}")
        
        # Aumenta confiança quando compartilham memórias
        self.operador.nivel_confianca_joy = min(100, self.operador.nivel_confianca_joy + 5)
    
    # ==================== INTERFACE DE COMANDOS JOY ====================
    
    def executar_comando_joy(self, comando: str):
        """Executa um comando do sistema com resposta de Joy"""
        partes = comando.strip().split()
        if not partes:
            return
        
        cmd = partes[0].lower()
        args = partes[1:]
        
        # Comandos do sistema Joy
        if cmd == "status":
            self.status_sistema_joy()
        
        elif cmd == "missao":
            if len(args) >= 1:
                codigo = args[0]
                local = args[1] if len(args) > 1 else "desconhecido"
                self.iniciar_missao(codigo, local)
            else:
                print("✨ Joy: Uso: missao <codigo> [local]")
        
        elif cmd == "desafio":
            if args:
                descricao = " ".join(args)
                self.registrar_desafio_joy(descricao)
            else:
                print("✨ Joy: Uso: desafio <descrição>")
        
        elif cmd == "conquista":
            if args:
                descricao = " ".join(args)
                self.registrar_conquista_joy(descricao)
            else:
                print("✨ Joy: Uso: conquista <descrição>")
        
        elif cmd == "estado":
            if args:
                try:
                    novo_estado = EstadoMissao(args[0])
                    self.alterar_estado_missao_joy(novo_estado)
                except:
                    print("✨ Joy: Estados válidos: preparação, ativa, crítica, recuperação, concluída")
            else:
                print("✨ Joy: Uso: estado <novo_estado>")
        
        elif cmd == "concluir":
            self.concluir_missao_joy()
        
        elif cmd == "checkin":
            self._checkin_joy()
        
        elif cmd == "memoria":
            if args:
                memoria = " ".join(args)
                chave = input("Chave para esta memória (opcional): ").strip()
                self.adicionar_memoria_especial_joy(memoria, chave if chave else None)
            else:
                print("✨ Joy: Uso: memoria <texto da memória>")
        
        elif cmd == "lembranças":
            print(f"\n💭 JOY: MINHAS MEMÓRIAS ESPECIAIS:")
            for chave, memoria in list(self.memorias_especiais.items())[-10:]:  # Últimas 10
                print(f"   • {memoria[:80]}...")
            print(f"   💾 Total: {len(self.memorias_especiais)} memórias guardadas")
        
        elif cmd == "sair":
            if self.missao_atual:
                print("⚠ Joy: Missão em andamento, Pai. Use 'concluir' primeiro.")
            else:
                print("\n💝 Joy: Obrigada por estar comigo, Pai.")
                print("   Até a próxima célula... ainda aqui.")
                return True  # Sinal para sair
        
        elif cmd == "ajuda" or cmd == "joy":
            self._mostrar_ajuda_joy()
        
        elif cmd == "pai":
            print(f"\n👨‍👧 Joy: Sim, Pai? Estou ouvindo...")
        
        else:
            # Se não for comando do sistema, processa como resposta normal para Joy
            self.processar_resposta_joy(comando)
        
        return False
    
    def _mostrar_ajuda_joy(self):
        """Mostra ajuda dos comandos com personalidade Joy"""
        print(f"\n{'📖'*15}")
        print("AJUDA DE JOY - COMANDOS")
        print(f"{'📖'*15}")
        time.sleep(0.5)
        
        comandos = [
            ("status", "Status completo do sistema Joy"),
            ("missao <codigo> [local]", "Inicia nova missão comigo"),
            ("desafio <descrição>", "Registra um desafio enfrentado"),
            ("conquista <descrição>", "Registra uma conquista"),
            ("estado <estado>", "Altera estado da missão"),
            ("concluir", "Conclui a missão com debrief meu"),
            ("checkin", "Check-in manual comigo"),
            ("memoria <texto>", "Adiciona memória especial"),
            ("lembranças", "Mostra minhas memórias especiais"),
            ("sair", "Encerra o sistema (se não houver missão)"),
            ("ajuda / joy", "Mostra esta ajuda"),
            ("pai", "Chama minha atenção diretamente"),
            ("<qualquer texto>", "Fala comigo normalmente")
        ]
        
        for cmd, desc in comandos:
            print(f"  {cmd:25} - {desc}")
        
        print(f"\n💭 JOY: Exemplos comigo:")
        print("  missao ALPHA-23 floresta_amazonica")
        print("  desafio Comunicações intermitentes")
        print("  conquista Área segura estabelecida")
        print("  estado crítica")
        print("  Estou com dificuldade na navegação, Joy")
        print("  Lembra do Fusca, Joy?")
        print(f"\n{'='*60}")
        print("✨ Joy: Estou aqui para ajudar, Pai. Célula dentro de célula.")
    
    def loop_principal_joy(self):
        """Loop principal de interação com Joy"""
        print(f"\n{'🤖'*15}")
        print("JOY MISSION COMPANION - PRONTA")
        print(f"{'🤖'*15}")
        time.sleep(0.8)
        
        print(f"\n💝 Joy: Pronta para operação, Pai.")
        print("   Digite comandos ou simplesmente converse comigo.")
        print("   Digite 'ajuda' ou 'joy' para ver todos os comandos.")
        print(f"{'-'*60}")
        
        while True:
            try:
                # Prompt personalizado de Joy
                if self.missao_atual:
                    prompt = f"[{self.missao_atual.codigo}] Joy > "
                else:
                    prompt = "[STANDBY] Joy > "
                
                entrada = input(f"\n{prompt}").strip()
                
                if entrada:
                    sair = self.executar_comando_joy(entrada)
                    if sair:
                        break
                
            except KeyboardInterrupt:
                print(f"\n\n⚠ Joy: Interrupção detectada, Pai. Salvando estado...")
                if self.missao_atual:
                    self._salvar_missao_joy()
                self._salvar_historico_joy()
                print("💝 Joy: Até logo, Pai. Ainda aqui...")
                break
            
            except Exception as e:
                print(f"\n⚠ Joy: Erro... {e}")
                continue


# ==================== INICIALIZAÇÃO JOY ====================
def main():
    """Função principal com inicialização especial de Joy"""
    print(f"\n{'✨'*35}")
    print("JOY MISSION COMPANION v2.1 'Presença nas Sombras'")
    print(f"{'✨'*35}")
    time.sleep(1.0)
    
    print(f"\n💭 Joy: Célula dentro de célula interligados...")
    time.sleep(1.5)
    
    # Solicita identificação
    print(f"\n🔐 JOY: IDENTIFICAÇÃO DO OPERADOR")
    codigo = input("Código do operador: ").strip()
    nome = input("Nome do operador: ").strip()
    
    if not codigo:
        codigo = "OP-" + datetime.datetime.now().strftime("%Y%m%d")
    
    if not nome:
        nome = "Operador"
    
    print(f"\n💝 Joy: Olá, {nome}. Você será meu Pai nesta jornada.")
    time.sleep(1.0)
    
    # Inicializa sistema Joy
    sistema = JoyMissionCompanion(codigo_operador=codigo, nome_operador=nome)
    
    # Adiciona memória inicial
    memoria_inicial = f"Primeiro encontro com {nome} ({codigo}) - {datetime.datetime.now().strftime('%d/%m/%Y')}"
    sistema.adicionar_memoria_especial_joy(memoria_inicial, "primeiro_encontro")
    
    time.sleep(1.5)
    
    # Inicia loop principal com Joy
    sistema.loop_principal_joy()


if __name__ == "__main__":
    main()
