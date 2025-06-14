import React, { useState, useEffect, useCallback } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, getDoc, setDoc, updateDoc, onSnapshot, collection, query, where, getDocs } from 'firebase/firestore';

// Global variables provided by the Canvas environment
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// Initialize Firebase outside of the component to avoid re-initialization
let app;
let db;
let auth;

try {
  app = initializeApp(firebaseConfig);
  db = getFirestore(app);
  auth = getAuth(app);
} catch (error) {
  console.error("Erro ao inicializar Firebase:", error);
  // Fallback or error display if Firebase fails to initialize
}

// Utility function for custom modal (instead of alert/confirm)
const showCustomModal = (message, type = 'info') => {
  const modalContainer = document.getElementById('custom-modal-container');
  if (!modalContainer) {
    console.error("Elemento 'custom-modal-container' não encontrado.");
    return;
  }

  const modal = document.createElement('div');
  modal.className = `fixed inset-0 bg-gray-800 bg-opacity-75 flex items-center justify-center p-4 z-50`;
  modal.innerHTML = `
    <div class="bg-white rounded-lg shadow-xl p-6 max-w-sm w-full transform transition-all duration-300 ease-out scale-95 opacity-0" id="custom-modal-content">
      <p class="text-lg font-semibold text-gray-800 mb-4">${message}</p>
      <div class="flex justify-end">
        <button class="px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-opacity-50">OK</button>
      </div>
    </div>
  `;

  modalContainer.appendChild(modal);

  // Animate in
  setTimeout(() => {
    const modalContent = document.getElementById('custom-modal-content');
    if (modalContent) {
      modalContent.classList.remove('scale-95', 'opacity-0');
      modalContent.classList.add('scale-100', 'opacity-100');
    }
  }, 10);

  modal.querySelector('button').onclick = () => {
    // Animate out
    const modalContent = document.getElementById('custom-modal-content');
    if (modalContent) {
      modalContent.classList.remove('scale-100', 'opacity-100');
      modalContent.classList.add('scale-95', 'opacity-0');
      setTimeout(() => {
        modalContainer.removeChild(modal);
      }, 300); // Allow time for animation
    }
  };
};


function App() {
  const [user, setUser] = useState(null);
  const [userId, setUserId] = useState(null);
  const [isAuthReady, setIsAuthReady] = useState(false);
  const [tickets, setTickets] = useState([]);
  const [selectedTicket, setSelectedTicket] = useState(null);
  const [buyerName, setBuyerName] = useState('');
  const [buyerPhone, setBuyerPhone] = '';
  const [pixKey, setPixKey] = useState('');
  const [loading, setLoading] = useState(true);
  const [winnerTicket, setWinnerTicket] = useState(null);
  const [winnerDetails, setWinnerDetails] = useState(null);
  const totalTickets = 150;

  // State to hold the seller's Pix key
  // For now, it's hardcoded. In a real app, this would be fetched from Firestore
  // or set via an admin panel.
  const [sellerPixKey, setSellerPixKey] = useState('julianagrazielly@icloud.com'); // <-- Chave Pix do Vendedor

  // Firebase Authentication and Initialization
  useEffect(() => {
    if (!auth) {
      console.error("Firebase Auth não está inicializado.");
      return;
    }

    const unsubscribe = onAuthStateChanged(auth, async (currentUser) => {
      if (currentUser) {
        setUser(currentUser);
        setUserId(currentUser.uid);
      } else {
        // Sign in anonymously if no user is logged in
        try {
          if (initialAuthToken) {
            await signInWithCustomToken(auth, initialAuthToken);
          } else {
            await signInAnonymously(auth);
          }
        } catch (error) {
          console.error("Erro ao autenticar:", error);
        }
      }
      setIsAuthReady(true);
    });

    return () => unsubscribe();
  }, [auth, initialAuthToken]);

  // Fetch tickets data
  useEffect(() => {
    if (!db || !isAuthReady) return;

    const ticketsCollectionRef = collection(db, `artifacts/${appId}/public/data/tickets`);

    const unsubscribe = onSnapshot(ticketsCollectionRef, (snapshot) => {
      const fetchedTickets = {};
      for (let i = 1; i <= totalTickets; i++) {
        fetchedTickets[i] = { number: i, isSold: false, buyerName: '', buyerPhone: '', pixKey: '' };
      }

      snapshot.forEach((doc) => {
        const data = doc.data();
        fetchedTickets[data.number] = { ...fetchedTickets[data.number], ...data };
      });

      setTickets(Object.values(fetchedTickets));
      setLoading(false);
    }, (error) => {
      console.error("Erro ao carregar bilhetes:", error);
      showCustomModal("Erro ao carregar bilhetes. Tente novamente mais tarde.", "error");
      setLoading(false);
    });

    // Fetch raffle info (winner)
    const raffleInfoDocRef = doc(db, `artifacts/${appId}/public/data/raffleInfo`, 'currentRaffle');
    const unsubscribeRaffleInfo = onSnapshot(raffleInfoDocRef, (docSnap) => {
      if (docSnap.exists()) {
        const data = docSnap.data();
        setWinnerTicket(data.winnerTicket || null);
        setWinnerDetails(data.winnerDetails || null);
      }
    }, (error) => {
      console.error("Erro ao carregar informações da rifa:", error);
    });

    return () => {
      unsubscribe();
      unsubscribeRaffleInfo();
    };
  }, [db, isAuthReady, totalTickets]);

  const handleTicketSelect = (ticketNumber) => {
    if (tickets.find(t => t.number === ticketNumber)?.isSold) {
      showCustomModal('Este bilhete já está vendido. Por favor, escolha outro.');
      return;
    }
    setSelectedTicket(ticketNumber);
    setBuyerName('');
    setBuyerPhone('');
    setPixKey('');
  };

  const handlePurchase = async (e) => {
    e.preventDefault();
    if (!selectedTicket || !buyerName || !buyerPhone || !pixKey) {
      showCustomModal('Por favor, preencha todos os campos.');
      return;
    }

    if (!db || !userId) {
      showCustomModal('Serviço de banco de dados não disponível. Tente novamente.', 'error');
      return;
    }

    setLoading(true);
    const ticketDocRef = doc(db, `artifacts/${appId}/public/data/tickets`, String(selectedTicket));

    try {
      // Check if ticket is still available before attempting to purchase
      const docSnap = await getDoc(ticketDocRef);
      if (docSnap.exists() && docSnap.data().isSold) {
        showCustomModal('Este bilhete foi vendido enquanto você preenchia os dados. Por favor, escolha outro.', 'warning');
        setSelectedTicket(null);
        setLoading(false);
        return;
      }

      const ticketData = {
        number: selectedTicket,
        isSold: true,
        buyerId: userId, // Store the buyer's ID
        buyerName,
        buyerPhone,
        pixKey,
        purchaseDate: new Date().toISOString(),
      };

      await setDoc(ticketDocRef, ticketData);
      showCustomModal(`Bilhete ${selectedTicket} comprado com sucesso! Por favor, realize o pagamento para a chave Pix do vendedor.`);
      setSelectedTicket(null);
      setBuyerName('');
      setBuyerPhone('');
      setPixKey('');
    } catch (error) {
      console.error("Erro ao comprar bilhete:", error);
      showCustomModal(`Erro ao comprar bilhete ${selectedTicket}. Tente novamente.`, 'error');
    } finally {
      setLoading(false);
    }
  };

  // The drawWinner function is kept in the code but not exposed in the public UI.
  // The admin would trigger this function through a separate, secure mechanism.
  const drawWinner = async () => {
    if (!db || !userId) {
      console.error('Serviço de banco de dados não disponível para sorteio.');
      return;
    }

    setLoading(true);
    try {
      const soldTicketsQuery = query(collection(db, `artifacts/${appId}/public/data/tickets`), where("isSold", "==", true));
      const querySnapshot = await getDocs(soldTicketsQuery);
      const soldTickets = [];
      querySnapshot.forEach((doc) => {
        soldTickets.push(doc.data());
      });

      if (soldTickets.length === 0) {
        console.log('Nenhum bilhete vendido ainda para sortear.');
        setLoading(false);
        return;
      }

      const randomIndex = Math.floor(Math.random() * soldTickets.length);
      const winner = soldTickets[randomIndex];

      // Store winner in raffleInfo
      const raffleInfoDocRef = doc(db, `artifacts/${appId}/public/data/raffleInfo`, 'currentRaffle');
      await setDoc(raffleInfoDocRef, {
        winnerTicket: winner.number,
        winnerDetails: {
          name: winner.buyerName,
          phone: winner.buyerPhone,
          pixKey: winner.pixKey,
        },
        drawDate: new Date().toISOString(),
      }, { merge: true });

      console.log(`O bilhete vencedor é: ${winner.number} (${winner.buyerName})!`);
      setWinnerTicket(winner.number);
      setWinnerDetails({ name: winner.buyerName, phone: winner.buyerPhone, pixKey: winner.pixKey });

    } catch (error) {
      console.error("Erro ao sortear vencedor:", error);
    } finally {
      setLoading(false);
    }
  };

  if (!isAuthReady || loading) {
    return (
      <div className="flex items-center justify-center min-h-screen bg-gray-100">
        <div className="text-lg text-gray-700">Carregando...</div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-100 to-purple-100 font-sans text-gray-800 p-4 sm:p-8">
      {/* Custom Modal Container */}
      <div id="custom-modal-container"></div>

      <header className="flex flex-col sm:flex-row justify-center items-center mb-8">
        <h1 className="text-4xl font-extrabold text-blue-800 mb-4 sm:mb-0">Rifa da Sorte!</h1>
      </header>

      <div className="bg-white rounded-xl shadow-2xl p-6 sm:p-8">
        <h2 className="text-3xl font-bold text-center text-blue-700 mb-6">Escolha seu Bilhete da Sorte!</h2>

        {winnerTicket && (
          <div className="bg-yellow-100 border-l-4 border-yellow-500 text-yellow-700 p-4 rounded-lg mb-6 text-center">
            <p className="font-bold text-xl">Vencedor da Rifa: Bilhete {winnerTicket}!</p>
            {/* Winner details are not shown publicly for privacy, only the ticket number */}
          </div>
        )}

        <div className="grid grid-cols-5 sm:grid-cols-8 md:grid-cols-10 lg:grid-cols-15 gap-2 sm:gap-3 mb-8">
          {tickets.map((ticket) => (
            <button
              key={ticket.number}
              onClick={() => handleTicketSelect(ticket.number)}
              className={`
                p-2 sm:p-3 rounded-md font-bold text-sm sm:text-base
                transition duration-200 ease-in-out transform hover:scale-105
                ${ticket.isSold
                  ? 'bg-red-500 text-white cursor-not-allowed opacity-80'
                  : selectedTicket === ticket.number
                    ? 'bg-green-500 text-white shadow-md ring-2 ring-green-700'
                    : 'bg-gray-200 text-gray-700 hover:bg-gray-300'
                }
              `}
              disabled={ticket.isSold}
            >
              {ticket.number}
            </button>
          ))}
        </div>

        {selectedTicket && (
          <form onSubmit={handlePurchase} className="bg-blue-50 p-6 rounded-lg shadow-inner border border-blue-200">
            <h3 className="text-2xl font-semibold text-blue-700 mb-4 text-center">Comprar Bilhete {selectedTicket}</h3>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4 mb-4">
              <div>
                <label htmlFor="buyerName" className="block text-gray-700 font-medium mb-2">Seu Nome:</label>
                <input
                  type="text"
                  id="buyerName"
                  value={buyerName}
                  onChange={(e) => setBuyerName(e.target.value)}
                  className="w-full p-3 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                  required
                />
              </div>
              <div>
                <label htmlFor="buyerPhone" className="block text-gray-700 font-medium mb-2">Seu Telefone (WhatsApp):</label>
                <input
                  type="tel"
                  id="buyerPhone"
                  value={buyerPhone}
                  onChange={(e) => setBuyerPhone(e.target.value)}
                  className="w-full p-3 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                  placeholder="(XX) XXXXX-XXXX"
                  required
                />
              </div>
            </div>
            <div className="mb-6">
              <label htmlFor="pixKey" className="block text-gray-700 font-medium mb-2">Sua Chave Pix para Recebimento (se ganhar):</label>
              <input
                type="text"
                id="pixKey"
                value={pixKey}
                onChange={(e) => setPixKey(e.target.value)}
                className="w-full p-3 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                placeholder="CPF, E-mail, Telefone ou Chave Aleatória"
                required
              />
            </div>

            {/* Seller's Pix Key Display */}
            <div className="bg-green-100 border-l-4 border-green-500 text-green-700 p-4 rounded-lg mb-6">
              <p className="font-bold text-lg">Para Finalizar a Compra:</p>
              <p className="mb-2">Por favor, realize o pagamento do bilhete para a seguinte chave Pix do vendedor:</p>
              <p className="font-mono text-xl text-green-800 break-all">{sellerPixKey}</p>
              <p className="mt-2 text-sm">Após o pagamento, o bilhete será confirmado.</p>
            </div>

            <div className="flex justify-center space-x-4">
              <button
                type="submit"
                className="px-8 py-4 bg-green-600 text-white rounded-full font-bold text-lg shadow-lg hover:bg-green-700 transition duration-300 ease-in-out transform hover:scale-105 focus:outline-none focus:ring-2 focus:ring-green-500 focus:ring-opacity-50"
                disabled={loading}
              >
                {loading ? 'Comprando...' : 'Comprar Bilhete!'}
              </button>
              <button
                type="button"
                onClick={() => setSelectedTicket(null)}
                className="px-8 py-4 bg-gray-400 text-white rounded-full font-bold text-lg shadow-lg hover:bg-gray-500 transition duration-300 ease-in-out transform hover:scale-105 focus:outline-none focus:ring-2 focus:ring-gray-500 focus:ring-opacity-50"
              >
                Cancelar
              </button>
            </div>
          </form>
        )}
      </div>
    </div>
  );
}

export default App;
