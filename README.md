import React, { useState, useMemo, useRef, useEffect } from 'react';
import { 
  ShoppingCart, 
  Trash2, 
  Plus, 
  Edit2, 
  Save, 
  Lock, 
  User, 
  MessageCircle 
} from 'lucide-react';
import { initializeApp } from 'firebase/app';
import { 
  getFirestore, 
  collection, 
  doc, 
  onSnapshot, 
  updateDoc, 
  setDoc,
} from 'firebase/firestore';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';

// --- CONFIGURAÇÃO FIREBASE ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'hal-studio-unique';

// --- CREDENCIAIS ADMINISTRATIVAS ---
const ADMIN_USER = "HALJORDANSTUDIO";
const ADMIN_PASS = "Maiteluna2105@";
const WHATSAPP_NUMBER = "5531984418439";

// Função para gerar link dinâmico do WhatsApp
const buildWhatsAppLink = (cart = []) => {
    if (cart.length === 0) {
        return `https://wa.me/${WHATSAPP_NUMBER}?text=${encodeURIComponent("Olá! Vi a vitrine da HAL STUDIO e gostaria de mais informações.")}`;
    }
    const itemsList = cart.map(item => `- ${item.name} (R$ ${item.price})`).join('%0A');
    const total = cart.reduce((acc, item) => acc + item.price, 0).toFixed(2);
    const message = `Olá! Gostaria de encomendar os seguintes itens da HAL STUDIO:%0A%0A${itemsList}%0A%0ATotal: R$ ${total}%0A%0AAguardando confirmação!`;
    return `https://wa.me/${WHATSAPP_NUMBER}?text=${message}`;
};

// Overlay estilo monitor antigo (CRT)
const CRTOverlay = () => (
    <>
        <div className="pointer-events-none fixed inset-0 z-[100] opacity-[0.03] select-none" 
             style={{ background: 'linear-gradient(rgba(18, 16, 16, 0) 50%, rgba(0, 0, 0, 0.25) 50%), linear-gradient(90deg, rgba(255, 0, 0, 0.06), rgba(0, 255, 0, 0.02), rgba(0, 0, 255, 0.06))', backgroundSize: '100% 2px, 3px 100%' }}></div>
        <div className="pointer-events-none fixed inset-0 z-[99] bg-gradient-to-b from-transparent via-transparent to-black/20"></div>
    </>
);

const Header = ({ currentPath, setPath, isAdmin, onLogin }) => {
    const NAV = [
        { id: "/", label: "Galeria" },
        { id: "/portfolio", label: "Projetos" },
        { id: "/contato", label: "Contato" },
    ];

    return (
        <header className="relative z-[70] border-b-4 border-[#FFCC00] bg-[#050505]/95 backdrop-blur-sm">
            <div className="max-w-[1600px] mx-auto px-6 py-4 flex items-center justify-between">
                <div onClick={() => setPath('/')} className="flex items-center gap-3 group cursor-pointer">
                    <div className="w-10 h-10 bg-[#FF3333] flex items-center justify-center border-b-4 border-[#990000]">
                        <span className="font-black text-2xl text-white italic">H</span>
                    </div>
                    <div className="leading-none">
                        <div className="font-black text-2xl sm:text-3xl text-[#FFCC00] italic uppercase tracking-tighter">HAL STUDIO</div>
                        <div className="font-mono text-[10px] sm:text-xs text-[#00E5FF] uppercase font-bold tracking-widest italic">
                            {isAdmin ? "SESSÃO ADMIN ATIVA" : "Painel Principal"}
                        </div>
                    </div>
                </div>

                <nav className="flex items-center gap-2">
                    {NAV.map((n) => (
                        <button
                            key={n.id}
                            onClick={() => setPath(n.id)}
                            className={`font-black uppercase tracking-widest text-xs sm:text-sm px-3 py-2 transition italic ${
                                currentPath === n.id ? "bg-[#FF3333] text-white" : "text-white/60 hover:text-white"
                            }`}
                        >
                            {n.label}
                        </button>
                    ))}
                    {!isAdmin && (
                        <button onClick={onLogin} className="p-2 text-[#FFCC00] hover:bg-white/5 rounded transition-colors ml-4">
                            <Lock className="w-4 h-4" />
                        </button>
                    )}
                </nav>
            </div>
        </header>
    );
};

export default function App() {
  const [user, setUser] = useState(null);
  const [products, setProducts] = useState([]);
  const [currentPath, setCurrentPath] = useState('/');
  const [activeCategory, setActiveCategory] = useState("TODOS");
  const [cart, setCart] = useState([]);
  const [selectedId, setSelectedId] = useState(null);
  const [isAdmin, setIsAdmin] = useState(false);
  const [isLoginOpen, setIsLoginOpen] = useState(false);
  const [loginInput, setLoginInput] = useState({ user: "", pass: "" });
  const [editingProduct, setEditingProduct] = useState(null);

  // 1. Inicialização de Autenticação
  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (e) {
        console.error("Erro na autenticação:", e);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, (u) => { if (u) setUser(u); });
    return () => unsubscribe();
  }, []);

  // 2. Sincronização de Dados com 40 slots iniciais
  useEffect(() => {
    if (!user) return;
    const productsCol = collection(db, 'artifacts', appId, 'public', 'data', 'products');
    
    const unsubscribe = onSnapshot(productsCol, (snapshot) => {
      if (snapshot.empty) {
        // Criar 40 produtos iniciais
        const initialProducts = Array.from({ length: 40 }).map((_, i) => {
            const id = (i + 1).toString();
            let cat = "ACTION FIGURES";
            if (i >= 20) cat = "BUSTOS";
            if (i >= 35) cat = "CHAVEIROS";

            return {
                id,
                name: `PRODUTO #${id.padStart(2, '0')}`,
                price: 150 + (i * 25),
                category: cat,
                image: `https://picsum.photos/seed/hal${id}/600/800`,
                desc: "Escultura premium produzida pela HAL STUDIO com acabamento em resina de alta qualidade."
            };
        });
        
        initialProducts.forEach(p => setDoc(doc(productsCol, p.id), p));
      } else {
        const data = snapshot.docs.map(d => ({ ...d.data() }));
        // Ordenar por ID para manter a organização
        const sortedData = data.sort((a, b) => parseInt(a.id) - parseInt(b.id));
        setProducts(sortedData);
        if (!selectedId && sortedData.length > 0) setSelectedId(sortedData[0].id);
      }
    }, (err) => console.error("Erro ao ler produtos:", err));
    
    return () => unsubscribe();
  }, [user]);

  const activeProduct = useMemo(() => 
    products.find(p => p.id === selectedId) || products[0], [products, selectedId]
  );

  const filteredProducts = useMemo(() => {
    return activeCategory === "TODOS" ? products : products.filter(p => p.category === activeCategory);
  }, [products, activeCategory]);

  const handleUpdateProduct = async (e) => {
    e.preventDefault();
    if (!isAdmin || !user) return;
    const productRef = doc(db, 'artifacts', appId, 'public', 'data', 'products', editingProduct.id);
    try {
        await updateDoc(productRef, {
            price: Number(editingProduct.price),
            image: editingProduct.image,
            name: editingProduct.name,
            category: editingProduct.category
        });
        setEditingProduct(null);
    } catch (err) { console.error("Erro ao atualizar produto:", err); }
  };

  const handleLogin = (e) => {
    e.preventDefault();
    if (loginInput.user === ADMIN_USER && loginInput.pass === ADMIN_PASS) {
        setIsAdmin(true);
        setIsLoginOpen(false);
        setLoginInput({ user: "", pass: "" });
    } else {
        alert("Credenciais Administrativas Inválidas.");
    }
  };

  const addToCart = (product) => {
    setCart([...cart, { ...product, cartId: Date.now() }]);
  };

  const removeFromCart = (cartId) => {
    setCart(cart.filter(item => item.cartId !== cartId));
  };

  if (!products.length) {
    return <div className="min-h-screen bg-black flex items-center justify-center text-[#FFCC00] font-black italic animate-pulse tracking-widest uppercase">Acedendo ao Sistema HAL STUDIO...</div>;
  }

  return (
    <div className="min-h-screen bg-[#050505] text-white font-sans overflow-x-hidden selection:bg-[#FFCC00] selection:text-black">
      <CRTOverlay />
      <Header currentPath={currentPath} setPath={setCurrentPath} isAdmin={isAdmin} onLogin={() => setIsLoginOpen(true)} />

      {/* LETREIRO ANIMADO */}
      <div className="bg-[#FFCC00] text-black py-2 border-y-2 border-black overflow-hidden font-black italic uppercase tracking-widest flex whitespace-nowrap z-30 relative">
        <div className="animate-marquee flex gap-12">
            {[1,2,3,4].map(i => <span key={i}>★ NOVOS PREÇOS ATUALIZADOS ★ ENCOMENDAS VIA WHATSAPP ★ RESINA PREMIUM ★ ENVIO NACIONAL ★ 40+ SLOTS DISPONÍVEIS ★</span>)}
        </div>
      </div>

      <main className="max-w-[1600px] mx-auto px-6 py-10">
        {currentPath === '/' && (
          <div className="grid grid-cols-1 lg:grid-cols-12 gap-12 animate-in fade-in duration-700">
            
            {/* PAINEL DE DESTAQUE FIXO */}
            <div className="lg:col-span-5">
              <div className="sticky top-10">
                <div className="relative aspect-[3/4] rounded-lg overflow-hidden border-4 border-neutral-800 bg-neutral-900 group shadow-2xl">
                  <img src={activeProduct?.image} className="w-full h-full object-cover transition-all duration-700" alt={activeProduct?.name} />
                  <div className="absolute inset-0 bg-gradient-to-t from-black via-transparent to-transparent opacity-90"></div>
                  
                  {isAdmin && (
                    <button 
                        onClick={() => setEditingProduct(activeProduct)}
                        className="absolute top-4 left-4 bg-blue-600 p-4 rounded-full hover:scale-110 transition-transform z-20 border-2 border-white shadow-lg"
                    >
                        <Edit2 className="w-5 h-5" />
                    </button>
                  )}

                  <div className="absolute bottom-0 left-0 right-0 p-8">
                    <span className="bg-[#FFCC00] text-black px-2 py-0.5 text-[10px] font-black uppercase italic tracking-widest">{activeProduct?.category}</span>
                    <h1 className="text-5xl font-black italic uppercase mt-2 mb-4 tracking-tighter leading-none">{activeProduct?.name}</h1>
                    <div className="text-4xl font-black text-[#FFCC00] italic mb-6">
                        R$ {activeProduct?.price?.toLocaleString('pt-BR', { minimumFractionDigits: 2 })}
                    </div>
                    <button 
                        onClick={() => addToCart(activeProduct)}
                        className="w-full bg-white text-black font-black py-5 uppercase italic tracking-widest hover:bg-[#FFCC00] transition-all flex items-center justify-center gap-3 active:scale-95 shadow-[8px_8px_0px_rgba(255,51,51,1)]"
                    >
                        <Plus className="w-5 h-5" /> Adicionar ao Carrinho
                    </button>
                  </div>
                </div>
              </div>
            </div>

            {/* LISTAGEM DE PRODUTOS (GRELHA DE 40+) */}
            <div className="lg:col-span-7 space-y-10">
               
               {/* CATEGORIAS */}
               <div className="flex flex-wrap gap-2">
                  {["TODOS", "ACTION FIGURES", "BUSTOS", "CHAVEIROS"].map(cat => (
                    <button
                      key={cat}
                      onClick={() => setActiveCategory(cat)}
                      className={`px-4 py-2 font-black italic uppercase text-xs border-b-4 transition-all tracking-widest ${
                        activeCategory === cat ? 'bg-[#FFCC00] text-black border-white translate-y-[-2px]' : 'bg-neutral-900 text-neutral-500 border-neutral-800 hover:text-white'
                      }`}
                    >
                      {cat}
                    </button>
                  ))}
               </div>

               {/* GRELHA DINÂMICA */}
               <div className="grid grid-cols-2 sm:grid-cols-3 xl:grid-cols-4 gap-4 pb-12 border-b border-neutral-800">
                  {filteredProducts.map(p => (
                    <div 
                      key={p.id}
                      onClick={() => setSelectedId(p.id)}
                      className={`group relative aspect-[3/4] cursor-pointer rounded border-2 transition-all duration-300 overflow-hidden ${selectedId === p.id ? 'border-[#FFCC00] scale-[1.05] shadow-[0_0_20px_rgba(255,204,0,0.3)] z-10' : 'border-neutral-800 opacity-70 hover:opacity-100 hover:border-neutral-600'}`}
                    >
                       <img src={p.image} className="w-full h-full object-cover transition-transform group-hover:scale-110" alt={p.name} />
                       <div className="absolute inset-0 bg-gradient-to-t from-black/90 via-transparent to-transparent opacity-80"></div>
                       <div className="absolute bottom-0 left-0 right-0 p-3">
                          <div className="text-[10px] font-black italic uppercase truncate text-white mb-0.5">{p.name}</div>
                          <div className="text-[#FFCC00] font-black text-xs italic">R$ {p.price}</div>
                       </div>
                    </div>
                  ))}
               </div>

               {/* CARRINHO DE COMPRAS INTEGRADO */}
               <div id="carrinho" className="bg-neutral-900/40 p-8 border-2 border-neutral-800 rounded-lg backdrop-blur-md relative overflow-hidden shadow-2xl">
                    <div className="absolute top-0 left-0 w-1 h-full bg-[#FF3333]"></div>
                    <div className="flex items-center justify-between mb-8">
                        <h3 className="font-black italic uppercase text-2xl flex items-center gap-3">
                            <ShoppingCart className="w-6 h-6 text-[#00E5FF]" /> 
                            Pedido HAL STUDIO 
                            <span className="bg-white text-black px-2 py-0.5 text-xs ml-2 rounded-sm">{cart.length}</span>
                        </h3>
                    </div>

                    {cart.length === 0 ? (
                        <div className="py-10 text-center border-2 border-dashed border-neutral-800 rounded text-neutral-600 font-black italic uppercase tracking-widest">
                            Seu carrinho está vazio. Selecione peças da galeria acima.
                        </div>
                    ) : (
                        <div className="space-y-4 mb-8 max-h-[500px] overflow-y-auto pr-2 custom-scrollbar">
                            {cart.map(item => (
                                <div key={item.cartId} className="flex justify-between items-center bg-black/40 p-4 rounded border border-neutral-800 group hover:border-[#FFCC00] transition-colors">
                                    <div className="flex items-center gap-4">
                                        <div className="w-12 h-12 bg-neutral-800 rounded overflow-hidden">
                                            <img src={item.image} className="w-full h-full object-cover" alt="" />
                                        </div>
                                        <div>
                                            <div className="font-black italic uppercase text-xs tracking-tight text-white">{item.name}</div>
                                            <div className="text-[#FFCC00] font-black italic text-[10px]">R$ {item.price}</div>
                                        </div>
                                    </div>
                                    <button 
                                        onClick={(e) => { e.stopPropagation(); removeFromCart(item.cartId); }} 
                                        className="text-neutral-600 hover:text-red-500 transition-colors p-2"
                                    >
                                        <Trash2 className="w-5 h-5" />
                                    </button>
                                </div>
                            ))}
                        </div>
                    )}

                    <div className="flex flex-col sm:flex-row justify-between items-end sm:items-center gap-6 pt-8 border-t border-neutral-800">
                        <div>
                            <div className="text-[10px] font-black text-neutral-500 uppercase tracking-widest mb-1 italic">Investimento Total</div>
                            <div className="text-4xl font-black italic text-white leading-none">
                                R$ {cart.reduce((a, b) => a + b.price, 0).toLocaleString('pt-BR', { minimumFractionDigits: 2 })}
                            </div>
                        </div>
                        <a 
                            href={buildWhatsAppLink(cart)} 
                            target="_blank" 
                            rel="noopener noreferrer"
                            className={`w-full sm:w-auto px-10 py-5 font-black uppercase text-sm italic tracking-widest flex items-center justify-center gap-3 transition-all ${
                                cart.length > 0 
                                ? "bg-[#25D366] text-white shadow-[6px_6px_0px_white] hover:translate-y-[-2px] active:translate-y-0" 
                                : "bg-neutral-800 text-neutral-500 cursor-not-allowed pointer-events-none"
                            }`}
                        >
                            <MessageCircle className="w-5 h-5" /> 
                            Finalizar no WhatsApp
                        </a>
                    </div>
               </div>
            </div>
          </div>
        )}

        {/* PROJETOS / PORTFOLIO */}
        {currentPath === '/portfolio' && (
             <div className="py-20 text-center animate-in zoom-in-95 duration-500">
                <h2 className="text-6xl font-black italic uppercase mb-4 tracking-tighter">Projetos <span className="text-[#FFCC00]">Especiais</span></h2>
                <p className="text-neutral-500 font-bold uppercase tracking-[0.5em] mb-12 italic">Customizações exclusivas HAL STUDIO</p>
                <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
                   <div className="relative group overflow-hidden border-4 border-neutral-800 aspect-video rounded-lg">
                     <img src="https://images.unsplash.com/photo-1542751371-adc38448a05e?w=800" className="w-full h-full object-cover grayscale hover:grayscale-0 transition-all duration-1000" alt="Projeto Diorama" />
                   </div>
                   <div className="relative group overflow-hidden border-4 border-neutral-800 aspect-video rounded-lg">
                     <img src="https://images.unsplash.com/photo-1550745165-9bc0b252726f?w=800" className="w-full h-full object-cover grayscale hover:grayscale-0 transition-all duration-1000" alt="Projeto Retro" />
                   </div>
                </div>
             </div>
        )}

        {/* CONTATO */}
        {currentPath === '/contato' && (
          <div className="max-w-2xl mx-auto py-20 animate-in slide-in-from-bottom-10 duration-500">
             <div className="bg-neutral-900 p-10 border-4 border-[#FFCC00] rounded-sm relative shadow-[20px_20px_0px_#FF3333]">
                <h2 className="text-4xl font-black italic uppercase mb-10 border-b-2 border-neutral-800 pb-6">Fale Conosco</h2>
                <div className="space-y-6">
                    <a href={`https://wa.me/${WHATSAPP_NUMBER}`} target="_blank" className="block bg-black/50 p-8 rounded border border-neutral-800 hover:border-[#25D366] transition-colors group">
                        <div className="text-[10px] font-black text-neutral-500 uppercase mb-2 tracking-widest">Suporte Direto via WhatsApp</div>
                        <div className="text-[#25D366] font-mono text-2xl font-bold tracking-tighter">+55 31 98441-8439</div>
                    </a>
                    <div className="p-8 border border-neutral-800 bg-black/30 rounded">
                        <p className="text-neutral-400 font-bold uppercase text-xs italic tracking-widest">Localização:</p>
                        <p className="text-white font-black italic uppercase mt-1">HAL STUDIO HQ - Belo Horizonte, MG</p>
                    </div>
                </div>
             </div>
          </div>
        )}
      </main>

      {/* LOGIN ADMIN MODAL */}
      {isLoginOpen && (
        <div className="fixed inset-0 z-[200] bg-black/95 backdrop-blur-xl flex items-center justify-center p-6">
            <div className="bg-neutral-900 border-4 border-[#FFCC00] p-10 max-w-sm w-full shadow-[0_0_100px_rgba(255,204,0,0.2)]">
                <div className="flex flex-col items-center mb-8">
                    <div className="bg-[#FFCC00] p-4 rounded-full mb-4"><Lock className="text-black w-8 h-8" /></div>
                    <h2 className="text-2xl font-black italic uppercase text-white tracking-tighter text-center">Painel Administrativo</h2>
                </div>
                <form onSubmit={handleLogin} className="space-y-4">
                    <div className="relative">
                        <User className="absolute left-4 top-1/2 -translate-y-1/2 text-neutral-500 w-4 h-4" />
                        <input type="text" value={loginInput.user} onChange={(e) => setLoginInput({...loginInput, user: e.target.value})} placeholder="LOGIN" className="w-full bg-black border-2 border-neutral-800 p-4 pl-12 font-black text-white outline-none focus:border-[#FFCC00] uppercase text-xs tracking-widest" />
                    </div>
                    <div className="relative">
                        <Lock className="absolute left-4 top-1/2 -translate-y-1/2 text-neutral-500 w-4 h-4" />
                        <input type="password" value={loginInput.pass} onChange={(e) => setLoginInput({...loginInput, pass: e.target.value})} placeholder="SENHA" className="w-full bg-black border-2 border-neutral-800 p-4 pl-12 font-black text-[#FFCC00] outline-none focus:border-[#FFCC00] text-xs tracking-widest" />
                    </div>
                    <div className="flex gap-3 pt-4">
                        <button type="submit" className="flex-1 bg-[#FFCC00] text-black font-black py-4 uppercase italic tracking-widest hover:bg-white transition-colors">Entrar</button>
                        <button type="button" onClick={() => setIsLoginOpen(false)} className="px-6 bg-neutral-800 font-black uppercase italic text-neutral-500 hover:text-white">X</button>
                    </div>
                </form>
            </div>
        </div>
      )}

      {/* MODAL DE EDIÇÃO DE PRODUTO */}
      {editingProduct && (
        <div className="fixed inset-0 z-[200] bg-black/95 backdrop-blur-xl flex items-center justify-center p-6 overflow-y-auto">
            <form onSubmit={handleUpdateProduct} className="bg-neutral-900 border-4 border-blue-600 p-10 max-w-lg w-full relative">
                <h2 className="text-3xl font-black italic uppercase mb-10 text-blue-500 tracking-tighter">Editar Slot #{editingProduct.id}</h2>
                <div className="space-y-6">
                    <div>
                        <label className="text-[10px] font-black uppercase text-neutral-500 mb-2 block tracking-widest italic">Nome do Produto</label>
                        <input type="text" value={editingProduct.name} onChange={(e) => setEditingProduct({...editingProduct, name: e.target.value})} className="w-full bg-black border-2 border-neutral-800 p-4 font-black italic uppercase outline-none focus:border-blue-600" />
                    </div>
                    <div className="grid grid-cols-2 gap-4">
                        <div>
                            <label className="text-[10px] font-black uppercase text-neutral-500 mb-2 block tracking-widest italic">Preço (R$)</label>
                            <input type="number" value={editingProduct.price} onChange={(e) => setEditingProduct({...editingProduct, price: e.target.value})} className="w-full bg-black border-2 border-neutral-800 p-4 font-black text-[#FFCC00] text-xl outline-none focus:border-blue-600" />
                        </div>
                        <div>
                            <label className="text-[10px] font-black uppercase text-neutral-500 mb-2 block tracking-widest italic">Categoria</label>
                            <select 
                                value={editingProduct.category} 
                                onChange={(e) => setEditingProduct({...editingProduct, category: e.target.value})}
                                className="w-full bg-black border-2 border-neutral-800 p-4 font-black text-white outline-none focus:border-blue-600 appearance-none uppercase text-xs"
                            >
                                <option value="ACTION FIGURES">ACTION FIGURES</option>
                                <option value="BUSTOS">BUSTOS</option>
                                <option value="CHAVEIROS">CHAVEIROS</option>
                            </select>
                        </div>
                    </div>
                    <div>
                        <label className="text-[10px] font-black uppercase text-neutral-500 mb-2 block tracking-widest italic">URL da Imagem</label>
                        <textarea value={editingProduct.image} onChange={(e) => setEditingProduct({...editingProduct, image: e.target.value})} className="w-full bg-black border-2 border-neutral-800 p-4 font-mono text-[10px] h-24 outline-none focus:border-blue-600" />
                    </div>
                </div>
                <div className="flex gap-4 mt-12 pt-6 border-t border-neutral-800">
                    <button type="submit" className="flex-1 bg-blue-600 text-white font-black py-5 uppercase italic tracking-widest flex items-center justify-center gap-3">
                        <Save className="w-5 h-5" /> Salvar Alterações
                    </button>
                    <button type="button" onClick={() => setEditingProduct(null)} className="px-8 bg-neutral-800 font-black uppercase italic text-neutral-500">Cancelar</button>
                </div>
            </form>
        </div>
      )}

      <style>{`
        @keyframes marquee { 0% { transform: translateX(0); } 100% { transform: translateX(-25%); } }
        .animate-marquee { animation: marquee 30s linear infinite; }
        .custom-scrollbar::-webkit-scrollbar { width: 4px; }
        .custom-scrollbar::-webkit-scrollbar-track { background: #000; }
        .custom-scrollbar::-webkit-scrollbar-thumb { background: #333; border-radius: 10px; }
        .custom-scrollbar::-webkit-scrollbar-thumb:hover { background: #FFCC00; }
      `}</style>
    </div>
  );
}
