import React, { useState, useEffect, createContext, useContext, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, doc, getDocs, getDoc, addDoc, setDoc, updateDoc, deleteDoc, onSnapshot, query, where } from 'firebase/firestore';

// Create a context for Firebase and user data
const AppContext = createContext(null);

// Custom Confirmation Modal Component
const ConfirmModal = ({ message, onConfirm, onCancel, onClose }) => {
    return (
        <div className="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-50 p-4">
            <div className="bg-white rounded-lg shadow-xl p-6 max-w-sm w-full text-center">
                <h3 className="text-xl font-semibold text-gray-800 mb-4">Confirm Action</h3>
                <p className="text-gray-700 mb-6">{message}</p>
                <div className="flex justify-center space-x-4">
                    <button
                        onClick={() => { onCancel(); onClose(); }}
                        className="bg-gray-500 hover:bg-gray-600 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md flex-1"
                    >
                        Cancel
                    </button>
                    <button
                        onClick={() => { onConfirm(); onClose(); }}
                        className="bg-red-600 hover:bg-red-700 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md flex-1"
                    >
                        Confirm
                    </button>
                </div>
            </div>
        </div>
    );
};

// Custom Message Modal Component (for alerts)
const MessageModal = ({ message, onClose }) => {
    return (
        <div className="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-50 p-4">
            <div className="bg-white rounded-lg shadow-xl p-6 max-w-sm w-full text-center">
                <h3 className="text-xl font-semibold text-gray-800 mb-4">Notification</h3>
                <p className="text-gray-700 mb-6">{message}</p>
                <button
                    onClick={onClose}
                    className="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md w-full"
                >
                    OK
                </button>
            </div>
        </div>
    );
};

// Utility function to format date
const formatDate = (timestamp) => {
    if (!timestamp) return '';
    const date = timestamp.toDate ? timestamp.toDate() : new Date(timestamp);
    return date.toLocaleDateString();
};

// Utility function to format currency
const formatCurrency = (amount) => {
    return new Intl.NumberFormat('en-US', {
        style: 'currency',
        currency: 'USD',
    }).format(amount);
};

// Global variables for Firebase config and app ID
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// AppProvider component to manage Firebase initialization and authentication
const AppProvider = ({ children }) => {
    const [db, setDb] = useState(null);
    const [auth, setAuth] = useState(null);
    const [userId, setUserId] = useState(null);
    const [loading, setLoading] = useState(true);
    const [isAuthReady, setIsAuthReady] = useState(false);

    useEffect(() => {
        try {
            const app = initializeApp(firebaseConfig);
            const firestore = getFirestore(app);
            const firebaseAuth = getAuth(app);

            setDb(firestore);
            setAuth(firebaseAuth);

            const unsubscribe = onAuthStateChanged(firebaseAuth, async (user) => {
                if (user) {
                    setUserId(user.uid);
                } else {
                    if (initialAuthToken) {
                        try {
                            await signInWithCustomToken(firebaseAuth, initialAuthToken);
                        } catch (error) {
                            console.error("Error signing in with custom token:", error);
                            await signInAnonymously(firebaseAuth);
                        }
                    } else {
                        await signInAnonymously(firebaseAuth);
                    }
                }
                setIsAuthReady(true);
                setLoading(false);
            });

            return () => unsubscribe();
        } catch (error) {
            console.error("Firebase initialization error:", error);
            setLoading(false);
        }
    }, []);

    if (loading || !isAuthReady) {
        return (
            <div className="flex items-center justify-center min-h-screen bg-gray-100">
                <div className="text-xl font-semibold text-gray-700">Loading application...</div>
            </div>
        );
    }

    return (
        <AppContext.Provider value={{ db, auth, userId, isAuthReady }}>
            {children}
        </AppContext.Provider>
    );
};

// --- Components ---

// Dashboard Component
const Dashboard = ({ navigate }) => {
    const { db, userId, isAuthReady } = useContext(AppContext);
    const [jobs, setJobs] = useState([]);
    const [customers, setCustomers] = useState([]);
    const [showConfirm, setShowConfirm] = useState(false);
    const [jobToDelete, setJobToDelete] = useState(null);
    const [message, setMessage] = useState('');
    const [showMessageModal, setShowMessageModal] = useState(false);

    useEffect(() => {
        if (!db || !userId || !isAuthReady) return;

        // Fetch jobs
        const jobsCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/jobs`);
        const unsubscribeJobs = onSnapshot(jobsCollectionRef, (snapshot) => {
            const jobsData = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
            setJobs(jobsData);
        }, (error) => {
            console.error("Error fetching jobs:", error);
            setMessage("Error fetching jobs.");
            setShowMessageModal(true);
        });

        // Fetch customers
        const customersCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/customers`);
        const unsubscribeCustomers = onSnapshot(customersCollectionRef, (snapshot) => {
            const customersData = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
            setCustomers(customersData);
        }, (error) => {
            console.error("Error fetching customers:", error);
            setMessage("Error fetching customers.");
            setShowMessageModal(true);
        });

        return () => {
            unsubscribeJobs();
            unsubscribeCustomers();
        };
    }, [db, userId, isAuthReady]);

    const handleDeleteJob = (jobId) => {
        setJobToDelete(jobId);
        setShowConfirm(true);
    };

    const confirmDelete = async () => {
        if (!db || !userId || !jobToDelete) return;
        try {
            await deleteDoc(doc(db, `artifacts/${appId}/users/${userId}/jobs`, jobToDelete));
            setMessage("Job deleted successfully!");
            setShowMessageModal(true);
        } catch (error) {
            console.error("Error deleting job:", error);
            setMessage("Error deleting job.");
            setShowMessageModal(true);
        } finally {
            setJobToDelete(null);
        }
    };

    const cancelDelete = () => {
        setJobToDelete(null);
    };

    const handleRestoreJob = async (jobId) => {
        if (!db || !userId) return;
        try {
            const jobRef = doc(db, `artifacts/${appId}/users/${userId}/jobs`, jobId);
            await updateDoc(jobRef, { status: 'active' });
            setMessage("Job restored successfully!");
            setShowMessageModal(true);
        } catch (error) {
            console.error("Error restoring job:", error);
            setMessage("Error restoring job.");
            setShowMessageModal(true);
        }
    };

    const activeJobs = jobs.filter(job => job.status === 'active');
    const completedJobs = jobs.filter(job => job.status === 'completed');
    const archivedJobs = jobs.filter(job => job.status === 'archived');

    return (
        <div className="p-4 sm:p-6 bg-gray-50 min-h-screen">
            <h2 className="text-2xl sm:text-3xl font-bold text-gray-800 mb-6">Dashboard</h2>

            <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 mb-6">
                <div className="bg-white p-5 rounded-lg shadow-md text-center">
                    <h3 className="text-lg font-semibold text-gray-700">Total Jobs</h3>
                    <p className="text-3xl font-bold text-blue-600">{jobs.length}</p>
                </div>
                <div className="bg-white p-5 rounded-lg shadow-md text-center">
                    <h3 className="text-lg font-semibold text-gray-700">Active Jobs</h3>
                    <p className="text-3xl font-bold text-green-600">{activeJobs.length}</p>
                </div>
                <div className="bg-white p-5 rounded-lg shadow-md text-center">
                    <h3 className="text-lg font-semibold text-gray-700">Total Customers</h3>
                    <p className="text-3xl font-bold text-purple-600">{customers.length}</p>
                </div>
            </div>

            <div className="bg-white p-4 sm:p-6 rounded-lg shadow-md mb-6">
                <div className="flex justify-between items-center mb-4">
                    <h3 className="text-xl font-semibold text-gray-800">Active Jobs</h3>
                    <button
                        onClick={() => navigate('jobDetail')}
                        className="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md text-sm sm:text-base"
                    >
                        + Add New Job
                    </button>
                </div>
                {activeJobs.length === 0 ? (
                    <p className="text-gray-600">No active jobs found.</p>
                ) : (
                    <div className="overflow-x-auto">
                        <table className="min-w-full divide-y divide-gray-200">
                            <thead className="bg-gray-50">
                                <tr>
                                    <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Job ID</th>
                                    <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Customer</th>
                                    <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Date</th>
                                    <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Status</th>
                                    <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Actions</th>
                                </tr>
                            </thead>
                            <tbody className="bg-white divide-y divide-gray-200">
                                {activeJobs.map(job => (
                                    <tr key={job.id}>
                                        <td className="px-3 py-2 whitespace-nowrap text-sm font-medium text-gray-900">{job.id.substring(0, 8)}</td>
                                        <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-700">{job.customerName}</td>
                                        <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-700">{formatDate(job.date)}</td>
                                        <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-700">{job.status}</td>
                                        <td className="px-3 py-2 whitespace-nowrap text-sm font-medium">
                                            <button
                                                onClick={() => navigate('jobDetail', { jobId: job.id })}
                                                className="text-blue-600 hover:text-blue-900 mr-2"
                                            >
                                                Edit
                                            </button>
                                            <button
                                                onClick={() => handleDeleteJob(job.id)}
                                                className="text-red-600 hover:text-red-900"
                                            >
                                                Archive
                                            </button>
                                        </td>
                                    </tr>
                                ))}
                            </tbody>
                        </table>
                    </div>
                )}
            </div>

            <div className="bg-white p-4 sm:p-6 rounded-lg shadow-md">
                <h3 className="text-xl font-semibold text-gray-800 mb-4">Archived Jobs</h3>
                {archivedJobs.length === 0 ? (
                    <p className="text-gray-600">No archived jobs found.</p>
                ) : (
                    <div className="overflow-x-auto">
                        <table className="min-w-full divide-y divide-gray-200">
                            <thead className="bg-gray-50">
                                <tr>
                                    <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Job ID</th>
                                    <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Customer</th>
                                    <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Date</th>
                                    <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Actions</th>
                                </tr>
                            </thead>
                            <tbody className="bg-white divide-y divide-gray-200">
                                {archivedJobs.map(job => (
                                    <tr key={job.id}>
                                        <td className="px-3 py-2 whitespace-nowrap text-sm font-medium text-gray-900">{job.id.substring(0, 8)}</td>
                                        <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-700">{job.customerName}</td>
                                        <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-700">{formatDate(job.date)}</td>
                                        <td className="px-3 py-2 whitespace-nowrap text-sm font-medium">
                                            <button
                                                onClick={() => handleRestoreJob(job.id)}
                                                className="text-green-600 hover:text-green-900 mr-2"
                                            >
                                                Restore
                                            </button>
                                            <button
                                                onClick={() => handleDeleteJob(job.id)} // Re-using for permanent delete (or modify logic)
                                                className="text-red-600 hover:text-red-900"
                                            >
                                                Delete
                                            </button>
                                        </td>
                                    </tr>
                                ))}
                            </tbody>
                        </table>
                    </div>
                )}
            </div>

            {showConfirm && (
                <ConfirmModal
                    message="Are you sure you want to archive/delete this job? This action cannot be undone."
                    onConfirm={confirmDelete}
                    onCancel={cancelDelete}
                    onClose={() => setShowConfirm(false)}
                />
            )}
            {showMessageModal && (
                <MessageModal
                    message={message}
                    onClose={() => setShowMessageModal(false)}
                />
            )}
        </div>
    );
};

// Customers Component
const Customers = ({ navigate }) => {
    const { db, userId, isAuthReady } = useContext(AppContext);
    const [customers, setCustomers] = useState([]);
    const [showConfirm, setShowConfirm] = useState(false);
    const [customerToDelete, setCustomerToDelete] = useState(null);
    const [message, setMessage] = useState('');
    const [showMessageModal, setShowMessageModal] = useState(false);

    useEffect(() => {
        if (!db || !userId || !isAuthReady) return;

        const customersCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/customers`);
        const unsubscribe = onSnapshot(customersCollectionRef, (snapshot) => {
            const customersData = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
            setCustomers(customersData);
        }, (error) => {
            console.error("Error fetching customers:", error);
            setMessage("Error fetching customers.");
            setShowMessageModal(true);
        });

        return () => unsubscribe();
    }, [db, userId, isAuthReady]);

    const handleDeleteCustomer = (customerId) => {
        setCustomerToDelete(customerId);
        setShowConfirm(true);
    };

    const confirmDelete = async () => {
        if (!db || !userId || !customerToDelete) return;
        try {
            await deleteDoc(doc(db, `artifacts/${appId}/users/${userId}/customers`, customerToDelete));
            setMessage("Customer deleted successfully!");
            setShowMessageModal(true);
        } catch (error) {
            console.error("Error deleting customer:", error);
            setMessage("Error deleting customer.");
            setShowMessageModal(true);
        } finally {
            setCustomerToDelete(null);
        }
    };

    const cancelDelete = () => {
        setCustomerToDelete(null);
    };

    return (
        <div className="p-4 sm:p-6 bg-gray-50 min-h-screen">
            <div className="flex justify-between items-center mb-6">
                <h2 className="text-2xl sm:text-3xl font-bold text-gray-800">Customers</h2>
                <button
                    onClick={() => navigate('addCustomer')}
                    className="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md text-sm sm:text-base"
                >
                    + Add New Customer
                </button>
            </div>

            {customers.length === 0 ? (
                <p className="text-gray-600">No customers found. Add a new customer to get started.</p>
            ) : (
                <div className="overflow-x-auto">
                    <table className="min-w-full divide-y divide-gray-200">
                        <thead className="bg-gray-50">
                            <tr>
                                <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Name</th>
                                <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Phone</th>
                                <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Email</th>
                                <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Address</th>
                                <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Actions</th>
                            </tr>
                        </thead>
                        <tbody className="bg-white divide-y divide-gray-200">
                            {customers.map(customer => (
                                <tr key={customer.id}>
                                    <td className="px-3 py-2 whitespace-nowrap text-sm font-medium text-gray-900">{customer.name}</td>
                                    <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-700">{customer.phone}</td>
                                    <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-700">{customer.email}</td>
                                    <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-700">{customer.address}</td>
                                    <td className="px-3 py-2 whitespace-nowrap text-sm font-medium">
                                        <button
                                            onClick={() => navigate('addCustomer', { customerId: customer.id })}
                                            className="text-blue-600 hover:text-blue-900 mr-2"
                                        >
                                            Edit
                                        </button>
                                        <button
                                            onClick={() => handleDeleteCustomer(customer.id)}
                                            className="text-red-600 hover:text-red-900"
                                        >
                                            Delete
                                        </button>
                                    </td>
                                </tr>
                            ))}
                        </tbody>
                    </table>
                </div>
            )}

            {showConfirm && (
                <ConfirmModal
                    message="Are you sure you want to delete this customer? This action cannot be undone."
                    onConfirm={confirmDelete}
                    onCancel={cancelDelete}
                    onClose={() => setShowConfirm(false)}
                />
            )}
            {showMessageModal && (
                <MessageModal
                    message={message}
                    onClose={() => setShowMessageModal(false)}
                />
            )}
        </div>
    );
};

// Add/Edit Customer Component
const AddCustomer = ({ navigate, customerId }) => {
    const { db, userId, isAuthReady } = useContext(AppContext);
    const [customer, setCustomer] = useState({
        name: '',
        phone: '',
        email: '',
        address: '',
    });
    const [message, setMessage] = useState('');
    const [showMessageModal, setShowMessageModal] = useState(false);

    useEffect(() => {
        if (!db || !userId || !isAuthReady) return;

        if (customerId) {
            const fetchCustomer = async () => {
                try {
                    const docRef = doc(db, `artifacts/${appId}/users/${userId}/customers`, customerId);
                    const docSnap = await getDoc(docRef);
                    if (docSnap.exists()) {
                        setCustomer(docSnap.data());
                    } else {
                        setMessage("Customer not found.");
                        setShowMessageModal(true);
                    }
                } catch (error) {
                    console.error("Error fetching customer:", error);
                    setMessage("Error fetching customer.");
                    setShowMessageModal(true);
                }
            };
            fetchCustomer();
        }
    }, [db, userId, isAuthReady, customerId]);

    const handleChange = (e) => {
        const { name, value } = e.target;
        setCustomer(prev => ({ ...prev, [name]: value }));
    };

    const handleSubmit = async (e) => {
        e.preventDefault();
        if (!db || !userId) return;

        try {
            if (customerId) {
                await updateDoc(doc(db, `artifacts/${appId}/users/${userId}/customers`, customerId), customer);
                setMessage("Customer updated successfully!");
            } else {
                await addDoc(collection(db, `artifacts/${appId}/users/${userId}/customers`), customer);
                setMessage("Customer added successfully!");
                setCustomer({ name: '', phone: '', email: '', address: '' }); // Clear form
            }
            setShowMessageModal(true);
            navigate('customers'); // Navigate back to customers list
        } catch (error) {
            console.error("Error saving customer:", error);
            setMessage("Error saving customer.");
            setShowMessageModal(true);
        }
    };

    return (
        <div className="p-4 sm:p-6 bg-gray-50 min-h-screen">
            <h2 className="text-2xl sm:text-3xl font-bold text-gray-800 mb-6">{customerId ? 'Edit Customer' : 'Add New Customer'}</h2>
            <form onSubmit={handleSubmit} className="bg-white p-4 sm:p-6 rounded-lg shadow-md space-y-4">
                <div>
                    <label htmlFor="name" className="block text-sm font-medium text-gray-700 mb-1">Name</label>
                    <input
                        type="text"
                        id="name"
                        name="name"
                        value={customer.name}
                        onChange={handleChange}
                        className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                        required
                    />
                </div>
                <div>
                    <label htmlFor="phone" className="block text-sm font-medium text-gray-700 mb-1">Phone</label>
                    <input
                        type="tel"
                        id="phone"
                        name="phone"
                        value={customer.phone}
                        onChange={handleChange}
                        className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                    />
                </div>
                <div>
                    <label htmlFor="email" className="block text-sm font-medium text-gray-700 mb-1">Email</label>
                    <input
                        type="email"
                        id="email"
                        name="email"
                        value={customer.email}
                        onChange={handleChange}
                        className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                    />
                </div>
                <div>
                    <label htmlFor="address" className="block text-sm font-medium text-gray-700 mb-1">Address</label>
                    <textarea
                        id="address"
                        name="address"
                        value={customer.address}
                        onChange={handleChange}
                        rows="3"
                        className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                    ></textarea>
                </div>
                <div className="flex flex-col sm:flex-row space-y-2 sm:space-y-0 sm:space-x-4">
                    <button
                        type="submit"
                        className="bg-green-600 hover:bg-green-700 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md flex-1"
                    >
                        {customerId ? 'Update Customer' : 'Add Customer'}
                    </button>
                    <button
                        type="button"
                        onClick={() => navigate('customers')}
                        className="bg-gray-500 hover:bg-gray-600 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md flex-1"
                    >
                        Cancel
                    </button>
                </div>
            </form>
            {showMessageModal && (
                <MessageModal
                    message={message}
                    onClose={() => setShowMessageModal(false)}
                />
            )}
        </div>
    );
};

// Customer Search Modal Component
const CustomerSearchModal = ({ onClose, onSelectCustomer }) => {
    const { db, userId, isAuthReady } = useContext(AppContext);
    const [customers, setCustomers] = useState([]);
    const [searchTerm, setSearchTerm] = useState('');
    const [message, setMessage] = useState('');
    const [showMessageModal, setShowMessageModal] = useState(false);

    useEffect(() => {
        if (!db || !userId || !isAuthReady) return;

        const customersCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/customers`);
        const unsubscribe = onSnapshot(customersCollectionRef, (snapshot) => {
            const customersData = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
            setCustomers(customersData);
        }, (error) => {
            console.error("Error fetching customers:", error);
            setMessage("Error fetching customers for search.");
            setShowMessageModal(true);
        });

        return () => unsubscribe();
    }, [db, userId, isAuthReady]);

    const filteredCustomers = customers.filter(customer =>
        customer.name.toLowerCase().includes(searchTerm.toLowerCase()) ||
        customer.phone.includes(searchTerm) ||
        customer.email.toLowerCase().includes(searchTerm.toLowerCase())
    );

    return (
        <div className="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-50 p-4">
            <div className="bg-white rounded-lg shadow-xl p-6 max-w-lg w-full max-h-[90vh] overflow-y-auto">
                <h3 className="text-xl font-semibold text-gray-800 mb-4">Search Customer</h3>
                <input
                    type="text"
                    placeholder="Search by name, phone, or email"
                    value={searchTerm}
                    onChange={(e) => setSearchTerm(e.target.value)}
                    className="mb-4 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                />
                <div className="max-h-60 overflow-y-auto border border-gray-200 rounded-md">
                    {filteredCustomers.length === 0 ? (
                        <p className="p-4 text-gray-600 text-center">No customers found.</p>
                    ) : (
                        <ul className="divide-y divide-gray-200">
                            {filteredCustomers.map(customer => (
                                <li
                                    key={customer.id}
                                    onClick={() => { onSelectCustomer(customer); onClose(); }}
                                    className="p-3 hover:bg-gray-100 cursor-pointer flex flex-col sm:flex-row sm:justify-between sm:items-center"
                                >
                                    <span className="font-medium text-gray-900">{customer.name}</span>
                                    <span className="text-sm text-gray-600">{customer.phone}</span>
                                </li>
                            ))}
                        </ul>
                    )}
                </div>
                <button
                    onClick={onClose}
                    className="mt-4 bg-gray-500 hover:bg-gray-600 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md w-full"
                >
                    Close
                </button>
            </div>
            {showMessageModal && (
                <MessageModal
                    message={message}
                    onClose={() => setShowMessageModal(false)}
                />
            )}
        </div>
    );
};

// Job Detail Component (Add/Edit Job)
const JobDetail = ({ navigate, jobId }) => {
    const { db, userId, isAuthReady } = useContext(AppContext);
    const [job, setJob] = useState({
        customerName: '',
        customerId: '',
        date: new Date().toISOString().split('T')[0],
        status: 'active',
        glassType: '',
        damageType: '',
        repairReplacement: '',
        cost: 0,
        quantity: 1,
        discountType: 'None',
        discountValue: 0,
        applySalesTax: true, // New field for sales tax application
        notes: '',
        totalAmount: 0, // This will be calculated and can be overridden
        paidAmount: 0,
    });
    const [customers, setCustomers] = useState([]);
    const [showCustomerSearch, setShowCustomerSearch] = useState(false);
    const [companyProfile, setCompanyProfile] = useState(null);
    const [message, setMessage] = useState('');
    const [showMessageModal, setShowMessageModal] = useState(false);
    const [showWorkOrderPreview, setShowWorkOrderPreview] = useState(false);

    useEffect(() => {
        if (!db || !userId || !isAuthReady) return;

        // Fetch company profile for sales tax rate and workflow options
        // Corrected Firestore path: artifacts/{appId}/users/{userId}/profile/companyData
        const companyProfileRef = doc(db, 'artifacts', appId, 'users', userId, 'profile', 'companyData');
        const unsubscribeProfile = onSnapshot(companyProfileRef, (docSnap) => {
            if (docSnap.exists()) {
                setCompanyProfile(docSnap.data());
            } else {
                // Ensure companyProfile is an object even if document doesn't exist
                setCompanyProfile({});
            }
        }, (error) => {
            console.error("Error fetching company profile:", error);
            setMessage("Error fetching company profile.");
            setShowMessageModal(true);
            // Ensure companyProfile is an object on error
            setCompanyProfile({});
        });

        // Fetch job details if editing
        if (jobId) {
            const fetchJob = async () => {
                try {
                    const docRef = doc(db, `artifacts/${appId}/users/${userId}/jobs`, jobId);
                    const docSnap = await getDoc(docRef);
                    if (docSnap.exists()) {
                        const jobData = docSnap.data();
                        setJob({
                            ...jobData,
                            date: jobData.date ? new Date(jobData.date.toDate()).toISOString().split('T')[0] : new Date().toISOString().split('T')[0],
                            cost: jobData.cost || 0,
                            quantity: jobData.quantity || 1,
                            discountValue: jobData.discountValue || 0,
                            applySalesTax: jobData.hasOwnProperty('applySalesTax') ? jobData.applySalesTax : true,
                            totalAmount: jobData.totalAmount || 0,
                            paidAmount: jobData.paidAmount || 0,
                        });
                    } else {
                        setMessage("Job not found.");
                        setShowMessageModal(true);
                    }
                } catch (error) {
                    console.error("Error fetching job:", error);
                    setMessage("Error fetching job.");
                    setShowMessageModal(true);
                }
            };
            fetchJob();
        }

        return () => unsubscribeProfile();
    }, [db, userId, isAuthReady, jobId]);

    const calculateServiceAmount = () => {
        let serviceCost = parseFloat(job.cost) || 0;
        let quantity = parseInt(job.quantity) || 1;
        let discountValue = parseFloat(job.discountValue) || 0;

        let subtotal = serviceCost * quantity;

        if (job.discountType === 'Percentage') {
            subtotal -= subtotal * (discountValue / 100);
        } else if (job.discountType === 'Flat Rate') {
            subtotal -= discountValue;
        }
        return Math.max(0, subtotal); // Ensure amount doesn't go negative
    };

    const calculateTotal = () => {
        const serviceAmount = calculateServiceAmount();
        const salesTaxRate = companyProfile?.salesTaxRate || 0;
        let taxAmount = 0;

        if (job.applySalesTax) {
            taxAmount = serviceAmount * (salesTaxRate / 100);
        }

        return serviceAmount + taxAmount;
    };

    const handleCustomerSelect = (selectedCustomer) => {
        setJob(prev => ({
            ...prev,
            customerId: selectedCustomer.id,
            customerName: selectedCustomer.name,
        }));
    };

    const handleChange = (e) => {
        const { name, value, type, checked } = e.target;
        setJob(prev => ({
            ...prev,
            [name]: type === 'checkbox' ? checked : value,
        }));
    };

    const handleNumberChange = (e) => {
        const { name, value } = e.target;
        // Allow empty string for number inputs, convert to 0 for calculations
        setJob(prev => ({
            ...prev,
            [name]: value === '' ? '' : parseFloat(value) || 0,
        }));
    };

    const handleSubmit = async (e) => {
        e.preventDefault();
        if (!db || !userId) return;

        // Calculate final total and paid amount before saving
        const finalTotal = calculateTotal();
        const finalPaid = parseFloat(job.paidAmount) || 0;

        const jobToSave = {
            ...job,
            date: new Date(job.date), // Convert date string to Firestore Timestamp
            cost: parseFloat(job.cost) || 0,
            quantity: parseInt(job.quantity) || 1,
            discountValue: parseFloat(job.discountValue) || 0,
            totalAmount: finalTotal, // Save the calculated total
            paidAmount: finalPaid,
        };

        try {
            if (jobId) {
                await updateDoc(doc(db, `artifacts/${appId}/users/${userId}/jobs`, jobId), jobToSave);
                setMessage("Job updated successfully!");
            } else {
                await addDoc(collection(db, `artifacts/${appId}/users/${userId}/jobs`), jobToSave);
                setMessage("Job added successfully!");
                // Clear form after adding new job
                setJob({
                    customerName: '',
                    customerId: '',
                    date: new Date().toISOString().split('T')[0],
                    status: 'active',
                    glassType: '',
                    damageType: '',
                    repairReplacement: '',
                    cost: 0,
                    quantity: 1,
                    discountType: 'None',
                    discountValue: 0,
                    applySalesTax: true,
                    notes: '',
                    totalAmount: 0,
                    paidAmount: 0,
                });
            }
            setShowMessageModal(true);
            navigate('dashboard'); // Navigate back to dashboard
        } catch (error) {
            console.error("Error saving job:", error);
            setMessage("Error saving job.");
            setShowMessageModal(true);
        }
    };

    const workflowOptions = companyProfile?.jobWorkflowOptions || {
        glassTypes: [],
        damageTypes: [],
        repairReplacementOptions: [],
    };

    const currentServiceAmount = calculateServiceAmount();
    const currentSalesTaxAmount = job.applySalesTax ? (currentServiceAmount * ((companyProfile?.salesTaxRate || 0) / 100)) : 0;
    const currentGrandTotal = currentServiceAmount + currentSalesTaxAmount;

    return (
        <div className="p-4 sm:p-6 bg-gray-50 min-h-screen">
            <h2 className="text-2xl sm:text-3xl font-bold text-gray-800 mb-6">{jobId ? 'Edit Job' : 'Add New Job'}</h2>
            <form onSubmit={handleSubmit} className="bg-white p-4 sm:p-6 rounded-lg shadow-md space-y-4">
                {/* Customer Details */}
                <div className="border-b pb-4 mb-4">
                    <h3 className="text-xl font-semibold text-gray-800 mb-3">Customer Details</h3>
                    <div>
                        <label htmlFor="customerName" className="block text-sm font-medium text-gray-700 mb-1">Customer</label>
                        <div className="flex space-x-2">
                            <input
                                type="text"
                                id="customerName"
                                name="customerName"
                                value={job.customerName}
                                onChange={handleChange}
                                className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                                placeholder="Select a customer"
                                readOnly
                                required
                            />
                            <button
                                type="button"
                                onClick={() => setShowCustomerSearch(true)}
                                className="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md text-sm sm:text-base flex-shrink-0"
                            >
                                Search
                            </button>
                        </div>
                    </div>
                </div>

                {/* Job Details */}
                <div className="border-b pb-4 mb-4">
                    <h3 className="text-xl font-semibold text-gray-800 mb-3">Job Details</h3>
                    <div>
                        <label htmlFor="date" className="block text-sm font-medium text-gray-700 mb-1">Date</label>
                        <input
                            type="date"
                            id="date"
                            name="date"
                            value={job.date}
                            onChange={handleChange}
                            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                            required
                        />
                    </div>
                    <div>
                        <label htmlFor="status" className="block text-sm font-medium text-gray-700 mb-1">Status</label>
                        <select
                            id="status"
                            name="status"
                            value={job.status}
                            onChange={handleChange}
                            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                        >
                            <option value="active">Active</option>
                            <option value="completed">Completed</option>
                            <option value="archived">Archived</option>
                        </select>
                    </div>
                </div>

                {/* Job Workflow Details */}
                <div className="border-b pb-4 mb-4">
                    <h3 className="text-xl font-semibold text-gray-800 mb-3">Job Workflow Details</h3>
                    <div>
                        <label htmlFor="glassType" className="block text-sm font-medium text-gray-700 mb-1">Glass Type</label>
                        <select
                            id="glassType"
                            name="glassType"
                            value={job.glassType}
                            onChange={handleChange}
                            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                        >
                            <option value="">Select Glass Type</option>
                            {workflowOptions.glassTypes.map((option, index) => (
                                <option key={index} value={option.name}>{option.name}</option>
                            ))}
                        </select>
                    </div>
                    <div>
                        <label htmlFor="damageType" className="block text-sm font-medium text-gray-700 mb-1">Damage Type</label>
                        <select
                            id="damageType"
                            name="damageType"
                            value={job.damageType}
                            onChange={handleChange}
                            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                        >
                            <option value="">Select Damage Type</option>
                            {workflowOptions.damageTypes.map((option, index) => (
                                <option key={index} value={option.name}>{option.name}</option>
                            ))}
                        </select>
                    </div>
                    <div>
                        <label htmlFor="repairReplacement" className="block text-sm font-medium text-gray-700 mb-1">Repair/Replacement</label>
                        <select
                            id="repairReplacement"
                            name="repairReplacement"
                            value={job.repairReplacement}
                            onChange={handleChange}
                            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                        >
                            <option value="">Select Option</option>
                            {workflowOptions.repairReplacementOptions.map((option, index) => (
                                <option key={index} value={option.name}>{option.name}</option>
                            ))}
                        </select>
                    </div>

                    {/* Cost, Quantity, Discount */}
                    <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
                        <div>
                            <label htmlFor="cost" className="block text-sm font-medium text-gray-700 mb-1">Cost ($)</label>
                            <input
                                type="number"
                                id="cost"
                                name="cost"
                                value={job.cost}
                                onChange={handleNumberChange}
                                className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                                min="0"
                                step="0.01"
                            />
                        </div>
                        <div>
                            <label htmlFor="quantity" className="block text-sm font-medium text-gray-700 mb-1">Quantity</label>
                            <input
                                type="number"
                                id="quantity"
                                name="quantity"
                                value={job.quantity}
                                onChange={handleNumberChange}
                                className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                                min="1"
                                step="1"
                            />
                        </div>
                    </div>
                    <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
                        <div>
                            <label htmlFor="discountType" className="block text-sm font-medium text-gray-700 mb-1">Discount Type</label>
                            <select
                                id="discountType"
                                name="discountType"
                                value={job.discountType}
                                onChange={handleChange}
                                className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                            >
                                <option value="None">None</option>
                                <option value="Percentage">Percentage (%)</option>
                                <option value="Flat Rate">Flat Rate ($)</option>
                            </select>
                        </div>
                        {job.discountType !== 'None' && (
                            <div>
                                <label htmlFor="discountValue" className="block text-sm font-medium text-gray-700 mb-1">Discount Value</label>
                                <input
                                    type="number"
                                    id="discountValue"
                                    name="discountValue"
                                    value={job.discountValue}
                                    onChange={handleNumberChange}
                                    className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                                    min="0"
                                    step="0.01"
                                />
                            </div>
                        )}
                    </div>
                    <div className="flex items-center mt-4">
                        <input
                            type="checkbox"
                            id="applySalesTax"
                            name="applySalesTax"
                            checked={job.applySalesTax}
                            onChange={handleChange}
                            className="h-4 w-4 text-blue-600 focus:ring-blue-500 border-gray-300 rounded"
                        />
                        <label htmlFor="applySalesTax" className="ml-2 block text-sm text-gray-900">Apply Sales Tax</label>
                    </div>
                </div>

                {/* Financial Breakdown */}
                <div className="border-b pb-4 mb-4">
                    <h3 className="text-xl font-semibold text-gray-800 mb-3">Financial Breakdown</h3>
                    <div className="space-y-2 text-gray-700">
                        <div className="flex justify-between">
                            <span>Material/Service Total:</span>
                            <span className="font-semibold">{formatCurrency(currentServiceAmount)}</span>
                        </div>
                        <div className="flex justify-between">
                            <span>Sales Tax ({companyProfile?.salesTaxRate || 0}%):</span>
                            <span className="font-semibold">{formatCurrency(currentSalesTaxAmount)}</span>
                        </div>
                        <div className="flex justify-between text-lg font-bold text-gray-900 border-t pt-2 mt-2">
                            <span>Grand Total:</span>
                            <span>{formatCurrency(currentGrandTotal)}</span>
                        </div>
                    </div>
                </div>

                {/* Notes */}
                <div>
                    <label htmlFor="notes" className="block text-sm font-medium text-gray-700 mb-1">Notes</label>
                    <textarea
                        id="notes"
                        name="notes"
                        value={job.notes}
                        onChange={handleChange}
                        rows="4"
                        className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                    ></textarea>
                </div>

                <div className="flex flex-col sm:flex-row space-y-2 sm:space-y-0 sm:space-x-4 mt-6">
                    <button
                        type="submit"
                        className="bg-green-600 hover:bg-green-700 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md flex-1"
                    >
                        {jobId ? 'Update Job' : 'Add Job'}
                    </button>
                    <button
                        type="button"
                        onClick={() => setShowWorkOrderPreview(true)}
                        className="bg-purple-600 hover:bg-purple-700 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md flex-1"
                    >
                        Preview Work Order
                    </button>
                    <button
                        type="button"
                        onClick={() => navigate('dashboard')}
                        className="bg-gray-500 hover:bg-gray-600 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md flex-1"
                    >
                        Cancel
                    </button>
                </div>
            </form>

            {showCustomerSearch && (
                <CustomerSearchModal
                    onClose={() => setShowCustomerSearch(false)}
                    onSelectCustomer={handleCustomerSelect}
                />
            )}
            {showMessageModal && (
                <MessageModal
                    message={message}
                    onClose={() => setShowMessageModal(false)}
                />
            )}
            {showWorkOrderPreview && (
                <WorkOrderPreviewModal
                    job={job}
                    companyProfile={companyProfile}
                    onClose={() => setShowWorkOrderPreview(false)}
                    onSaveJob={handleSubmit}
                    setJob={setJob} // Pass setJob to allow modification of total/paid
                />
            )}
        </div>
    );
};

// Work Order Preview Modal Component
const WorkOrderPreviewModal = ({ job, companyProfile, onClose, onSaveJob, setJob }) => {
    const { db, userId } = useContext(AppContext);
    const workOrderRef = useRef(null);
    const [previewJob, setPreviewJob] = useState({ ...job }); // Use a local state for editable fields

    useEffect(() => {
        // Recalculate total and paid if job details change
        const serviceAmount = calculateServiceAmount(previewJob, companyProfile);
        const salesTaxRate = companyProfile?.salesTaxRate || 0;
        const taxAmount = previewJob.applySalesTax ? (serviceAmount * (salesTaxRate / 100)) : 0;
        const calculatedTotal = serviceAmount + taxAmount;

        // Only update if the calculated total is different from the stored total,
        // or if it's a new job and totalAmount is 0.
        // This prevents infinite loops if the user manually sets totalAmount.
        if (previewJob.totalAmount === 0 || Math.abs(previewJob.totalAmount - calculatedTotal) > 0.01) {
             setPreviewJob(prev => ({
                ...prev,
                totalAmount: calculatedTotal,
                // If paidAmount is not set or is 0, default it to calculatedTotal
                paidAmount: prev.paidAmount === 0 ? calculatedTotal : prev.paidAmount
            }));
        }
    }, [job, companyProfile]); // Dependencies for recalculation

    const calculateServiceAmount = (currentJob, currentCompanyProfile) => {
        let serviceCost = parseFloat(currentJob.cost) || 0;
        let quantity = parseInt(currentJob.quantity) || 1;
        let discountValue = parseFloat(currentJob.discountValue) || 0;

        let subtotal = serviceCost * quantity;

        if (currentJob.discountType === 'Percentage') {
            subtotal -= subtotal * (discountValue / 100);
        } else if (currentJob.discountType === 'Flat Rate') {
            subtotal -= discountValue;
        }
        return Math.max(0, subtotal);
    };

    const handlePreviewChange = (e) => {
        const { name, value } = e.target;
        setPreviewJob(prev => ({
            ...prev,
            [name]: parseFloat(value) || 0, // Ensure numbers
        }));
    };

    const handleSaveAndClose = async () => {
        // Update the parent job state with the modified total and paid amounts
        setJob(prev => ({
            ...prev,
            totalAmount: previewJob.totalAmount,
            paidAmount: previewJob.paidAmount,
        }));
        await onSaveJob(); // Call the parent save function
        onClose();
    };

    const serviceAmount = calculateServiceAmount(previewJob, companyProfile);
    const salesTaxRate = companyProfile?.salesTaxRate || 0;
    const taxAmount = previewJob.applySalesTax ? (serviceAmount * (salesTaxRate / 100)) : 0;
    const balanceDue = (previewJob.totalAmount || 0) - (previewJob.paidAmount || 0);

    const generatePrintContent = () => {
        const content = workOrderRef.current;
        const printWindow = window.open('', '', 'height=600,width=800');
        printWindow.document.write('<html><head><title>Work Order</title>');
        printWindow.document.write('<link href="https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css" rel="stylesheet">');
        printWindow.document.write('<style>');
        printWindow.document.write(`
            body { font-family: 'Inter', sans-serif; margin: 20px; }
            .work-order-container { max-width: 800px; margin: 0 auto; padding: 20px; border: 1px solid #e2e8f0; border-radius: 8px; background-color: #fff; }
            h1, h2, h3 { color: #1a202c; }
            table { width: 100%; border-collapse: collapse; margin-top: 20px; }
            th, td { border: 1px solid #e2e8f0; padding: 8px; text-align: left; }
            th { background-color: #f7fafc; }
            .text-right { text-align: right; }
            .font-bold { font-weight: 700; }
            .mb-4 { margin-bottom: 1rem; }
            .mt-4 { margin-top: 1rem; }
            .text-xl { font-size: 1.25rem; }
            .text-2xl { font-size: 1.5rem; }
        `);
        printWindow.document.write('</style></head><body>');
        printWindow.document.write(content.innerHTML);
        printWindow.document.write('</body></html>');
        printWindow.document.close();
        printWindow.print();
    };

    const generateEmailSmsContent = (type) => {
        const companyName = companyProfile?.companyName || 'Auto Glass Service';
        const customerName = previewJob.customerName || 'Customer';
        const service = previewJob.repairReplacement || 'Auto Glass Service';
        const notes = previewJob.notes ? `\nNotes: ${previewJob.notes}` : '';
        const date = formatDate(previewJob.date);

        let financialDetails = `\n\nService Amount: ${formatCurrency(serviceAmount)}`;
        if (previewJob.applySalesTax) {
            financialDetails += `\nSales Tax (${salesTaxRate}%): ${formatCurrency(taxAmount)}`;
        }
        financialDetails += `\nTotal: ${formatCurrency(previewJob.totalAmount)}`;
        financialDetails += `\nPaid: ${formatCurrency(previewJob.paidAmount)}`;
        financialDetails += `\nBalance Due: ${formatCurrency(balanceDue)}`;

        if (type === 'email') {
            return `Subject: Your Work Order from ${companyName}\n\nDear ${customerName},\n\nHere is your work order summary for the ${service} completed on ${date}.${notes}${financialDetails}\n\nThank you for your business!\n${companyName}`;
        } else if (type === 'sms') {
            return `Work Order from ${companyName}:\n${service} on ${date}.${notes}\nTotal: ${formatCurrency(previewJob.totalAmount)}, Paid: ${formatCurrency(previewJob.paidAmount)}, Due: ${formatCurrency(balanceDue)}. Thank you!`;
        }
    };

    const copyToClipboard = (text) => {
        const textarea = document.createElement('textarea');
        textarea.value = text;
        document.body.appendChild(textarea);
        textarea.select();
        try {
            document.execCommand('copy');
            alert('Content copied to clipboard!'); // Using alert for simplicity, replace with custom modal if needed
        } catch (err) {
            console.error('Failed to copy: ', err);
            alert('Failed to copy content.');
        }
        document.body.removeChild(textarea);
    };

    return (
        <div className="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-50 p-4">
            <div className="bg-white rounded-lg shadow-xl p-4 sm:p-6 max-w-3xl w-full max-h-[95vh] overflow-y-auto">
                <h3 className="text-xl sm:text-2xl font-bold text-gray-800 mb-4">Work Order Preview</h3>

                <div ref={workOrderRef} className="p-4 border border-gray-200 rounded-md bg-gray-50 mb-4">
                    <h1 className="text-2xl sm:text-3xl font-bold text-center mb-4">{companyProfile?.companyName || 'Auto Glass Service'}</h1>
                    <p className="text-center text-gray-600 mb-6">{companyProfile?.address} | {companyProfile?.phone} | {companyProfile?.email}</p>

                    <div className="grid grid-cols-1 sm:grid-cols-2 gap-4 mb-6">
                        <div>
                            <h4 className="font-semibold text-gray-800">Customer Details:</h4>
                            <p className="text-gray-700">{previewJob.customerName}</p>
                            {/* In a real app, fetch customer details by ID */}
                        </div>
                        <div className="sm:text-right">
                            <p className="text-gray-700"><strong>Work Order ID:</strong> {job.id ? job.id.substring(0, 8) : 'N/A'}</p>
                            <p className="text-gray-700"><strong>Date:</strong> {formatDate(previewJob.date)}</p>
                            <p className="text-gray-700"><strong>Status:</strong> {previewJob.status}</p>
                        </div>
                    </div>

                    <h4 className="font-semibold text-gray-800 mb-3">Service Details:</h4>
                    <div className="overflow-x-auto mb-6">
                        <table className="min-w-full divide-y divide-gray-200">
                            <thead className="bg-gray-50">
                                <tr>
                                    <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Description</th>
                                    <th className="px-3 py-2 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">Qty</th>
                                    <th className="px-3 py-2 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">Cost/Unit</th>
                                    <th className="px-3 py-2 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">Discount</th>
                                    <th className="px-3 py-2 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">Amount</th>
                                </tr>
                            </thead>
                            <tbody className="bg-white divide-y divide-gray-200">
                                <tr>
                                    <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-900">
                                        {previewJob.repairReplacement}
                                        {previewJob.glassType && ` (${previewJob.glassType})`}
                                        {previewJob.damageType && ` - ${previewJob.damageType}`}
                                    </td>
                                    <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-700 text-right">{previewJob.quantity}</td>
                                    <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-700 text-right">{formatCurrency(previewJob.cost)}</td>
                                    <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-700 text-right">
                                        {previewJob.discountType === 'Percentage' ? `${previewJob.discountValue}%` : formatCurrency(previewJob.discountValue)}
                                    </td>
                                    <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-900 font-bold text-right">{formatCurrency(serviceAmount)}</td>
                                </tr>
                                {previewJob.notes && (
                                    <tr>
                                        <td colSpan="5" className="px-3 py-2 text-sm text-gray-700">
                                            <strong>Notes:</strong> {previewJob.notes}
                                        </td>
                                    </tr>
                                )}
                            </tbody>
                        </table>
                    </div>

                    <div className="flex flex-col items-end space-y-2 text-gray-800">
                        <div className="flex justify-between w-full sm:w-1/2">
                            <span>Subtotal:</span>
                            <span className="font-semibold">{formatCurrency(serviceAmount)}</span>
                        </div>
                        {previewJob.applySalesTax && (
                            <div className="flex justify-between w-full sm:w-1/2">
                                <span>Sales Tax ({salesTaxRate}%):</span>
                                <span className="font-semibold">{formatCurrency(taxAmount)}</span>
                            </div>
                        )}
                        <div className="flex justify-between w-full sm:w-1/2 text-lg font-bold border-t pt-2 mt-2">
                            <span>Total:</span>
                            <input
                                type="number"
                                name="totalAmount"
                                value={previewJob.totalAmount}
                                onChange={handlePreviewChange}
                                className="w-32 text-right px-2 py-1 border border-gray-300 rounded-md"
                                step="0.01"
                            />
                        </div>
                        <div className="flex justify-between w-full sm:w-1/2 text-lg font-bold">
                            <span>Paid:</span>
                            <input
                                type="number"
                                name="paidAmount"
                                value={previewJob.paidAmount}
                                onChange={handlePreviewChange}
                                className="w-32 text-right px-2 py-1 border border-gray-300 rounded-md"
                                step="0.01"
                            />
                        </div>
                        <div className="flex justify-between w-full sm:w-1/2 text-xl font-bold text-blue-700 border-t pt-2 mt-2">
                            <span>Balance Due:</span>
                            <span>{formatCurrency(balanceDue)}</span>
                        </div>
                    </div>
                </div>

                <div className="flex flex-col sm:flex-row space-y-2 sm:space-y-0 sm:space-x-4 mt-6">
                    <button
                        onClick={handleSaveAndClose}
                        className="bg-green-600 hover:bg-green-700 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md flex-1"
                    >
                        Save & Close
                    </button>
                    <button
                        onClick={generatePrintContent}
                        className="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md flex-1"
                    >
                        Print
                    </button>
                    <button
                        onClick={() => copyToClipboard(generateEmailSmsContent('email'))}
                        className="bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md flex-1"
                    >
                        Copy Email
                    </button>
                    <button
                        onClick={() => copyToClipboard(generateEmailSmsContent('sms'))}
                        className="bg-yellow-600 hover:bg-yellow-700 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md flex-1"
                    >
                        Copy SMS
                    </button>
                    <button
                        onClick={onClose}
                        className="bg-gray-500 hover:bg-gray-600 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md flex-1"
                    >
                        Close
                    </button>
                </div>
            </div>
        </div>
    );
};

// Settings Component
const Settings = ({ navigate }) => {
    const { db, userId, isAuthReady } = useContext(AppContext);
    const [profile, setProfile] = useState({
        companyName: '',
        address: '',
        phone: '',
        email: '',
        salesTaxRate: 0, // New field for sales tax rate
        jobWorkflowOptions: {
            glassTypes: [],
            damageTypes: [],
            repairReplacementOptions: [],
        },
    });
    const [message, setMessage] = useState('');
    const [showMessageModal, setShowMessageModal] = useState(false);

    useEffect(() => {
        if (!db || !userId || !isAuthReady) return;

        // Corrected Firestore path: artifacts/{appId}/users/{userId}/profile/companyData
        const profileRef = doc(db, 'artifacts', appId, 'users', userId, 'profile', 'companyData');
        const unsubscribe = onSnapshot(profileRef, (docSnap) => {
            if (docSnap.exists()) {
                setProfile(docSnap.data());
            } else {
                setProfile(prev => ({ // Keep existing defaults if no profile exists
                    ...prev,
                    companyName: '',
                    address: '',
                    phone: '',
                    email: '',
                    salesTaxRate: 0,
                    jobWorkflowOptions: {
                        glassTypes: [],
                        damageTypes: [],
                        repairReplacementOptions: [],
                    },
                }));
            }
        }, (error) => {
            console.error("Error fetching profile:", error);
            setMessage("Error fetching profile settings.");
            setShowMessageModal(true);
            setProfile(prev => ({ // Reset on error
                ...prev,
                companyName: '',
                address: '',
                phone: '',
                email: '',
                salesTaxRate: 0,
                jobWorkflowOptions: {
                    glassTypes: [],
                    damageTypes: [],
                    repairReplacementOptions: [],
                },
            }));
        });

        return () => unsubscribe();
    }, [db, userId, isAuthReady]);

    const handleChange = (e) => {
        const { name, value } = e.target;
        setProfile(prev => ({ ...prev, [name]: value }));
    };

    const handleNumberChange = (e) => {
        const { name, value } = e.target;
        setProfile(prev => ({ ...prev, [name]: parseFloat(value) || 0 }));
    };

    const handleSubmit = async (e) => {
        e.preventDefault();
        if (!db || !userId) return;

        try {
            // Corrected Firestore path: artifacts/{appId}/users/{userId}/profile/companyData
            await setDoc(doc(db, 'artifacts', appId, 'users', userId, 'profile', 'companyData'), profile, { merge: true });
            setMessage("Profile settings updated successfully!");
            setShowMessageModal(true);
        } catch (error) {
            console.error("Error updating profile:", error);
            setMessage("Error updating profile settings.");
            setShowMessageModal(true);
        }
    };

    return (
        <div className="p-4 sm:p-6 bg-gray-50 min-h-screen">
            <h2 className="text-2xl sm:text-3xl font-bold text-gray-800 mb-6">Settings</h2>
            <form onSubmit={handleSubmit} className="bg-white p-4 sm:p-6 rounded-lg shadow-md space-y-6">
                {/* Business Profile Settings */}
                <div className="border-b pb-4">
                    <h3 className="text-xl font-semibold text-gray-800 mb-4">Business Profile Settings</h3>
                    <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                        <div>
                            <label htmlFor="companyName" className="block text-sm font-medium text-gray-700 mb-1">Company Name</label>
                            <input
                                type="text"
                                id="companyName"
                                name="companyName"
                                value={profile.companyName}
                                onChange={handleChange}
                                className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                            />
                        </div>
                        <div>
                            <label htmlFor="address" className="block text-sm font-medium text-gray-700 mb-1">Address</label>
                            <input
                                type="text"
                                id="address"
                                name="address"
                                value={profile.address}
                                onChange={handleChange}
                                className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                            />
                        </div>
                        <div>
                            <label htmlFor="phone" className="block text-sm font-medium text-gray-700 mb-1">Phone</label>
                            <input
                                type="tel"
                                id="phone"
                                name="phone"
                                value={profile.phone}
                                onChange={handleChange}
                                className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                            />
                        </div>
                        <div>
                            <label htmlFor="email" className="block text-sm font-medium text-gray-700 mb-1">Email</label>
                            <input
                                type="email"
                                id="email"
                                name="email"
                                value={profile.email}
                                onChange={handleChange}
                                className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                            />
                        </div>
                    </div>
                    <button
                        type="submit"
                        className="mt-6 bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md w-full sm:w-auto"
                    >
                        Save Profile Settings
                    </button>
                </div>

                {/* Job Workflow Options */}
                <div className="border-b pb-4">
                    <h3 className="text-xl font-semibold text-gray-800 mb-4">Job Workflow Options</h3>
                    <button
                        type="button"
                        onClick={() => navigate('jobWorkflowSettings')}
                        className="bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md w-full sm:w-auto"
                    >
                        Edit Job Workflow Options
                    </button>
                </div>

                {/* Sales Tax Settings */}
                <div>
                    <h3 className="text-xl font-semibold text-gray-800 mb-4">Sales Tax Settings</h3>
                    <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                        <div>
                            <label htmlFor="salesTaxRate" className="block text-sm font-medium text-gray-700 mb-1">Sales Tax Rate (%)</label>
                            <input
                                type="number"
                                id="salesTaxRate"
                                name="salesTaxRate"
                                value={profile.salesTaxRate}
                                onChange={handleNumberChange}
                                className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                                min="0"
                                step="0.01"
                            />
                        </div>
                    </div>
                    <button
                        type="submit"
                        className="mt-6 bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md w-full sm:w-auto"
                    >
                        Save Sales Tax Settings
                    </button>
                </div>
            </form>
            {showMessageModal && (
                <MessageModal
                    message={message}
                    onClose={() => setShowMessageModal(false)}
                />
            )}
        </div>
    );
};

// Job Workflow Settings Component
const JobWorkflowSettings = ({ navigate }) => {
    const { db, userId, isAuthReady } = useContext(AppContext);
    const [profile, setProfile] = useState({
        jobWorkflowOptions: {
            glassTypes: [],
            damageTypes: [],
            repairReplacementOptions: [],
        },
    });
    const [currentOptionType, setCurrentOptionType] = useState('glassTypes'); // To manage which list is being edited
    const [newItem, setNewItem] = useState({ name: '', cost: '', discountType: 'None', discountValue: '', quantity: '' });
    const [editingIndex, setEditingIndex] = useState(null); // Index of item being edited
    const [message, setMessage] = useState('');
    const [showMessageModal, setShowMessageModal] = useState(false);

    useEffect(() => {
        if (!db || !userId || !isAuthReady) return;

        // Corrected Firestore path: artifacts/{appId}/users/{userId}/profile/companyData
        const profileRef = doc(db, 'artifacts', appId, 'users', userId, 'profile', 'companyData');
        const unsubscribe = onSnapshot(profileRef, (docSnap) => {
            if (docSnap.exists()) {
                setProfile(docSnap.data());
            } else {
                setProfile(prev => ({ // Keep existing defaults if no profile exists
                    ...prev,
                    jobWorkflowOptions: {
                        glassTypes: [],
                        damageTypes: [],
                        repairReplacementOptions: [],
                    },
                }));
            }
        }, (error) => {
            console.error("Error fetching profile for workflow:", error);
            setMessage("Error fetching workflow settings.");
            setShowMessageModal(true);
            setProfile(prev => ({ // Reset on error
                ...prev,
                jobWorkflowOptions: {
                    glassTypes: [],
                    damageTypes: [],
                    repairReplacementOptions: [],
                },
            }));
        });

        return () => unsubscribe();
    }, [db, userId, isAuthReady]);

    const handleNewItemChange = (e) => {
        const { name, value } = e.target;
        setNewItem(prev => ({ ...prev, [name]: value }));
    };

    const handleNewItemNumberChange = (e) => {
        const { name, value } = e.target;
        setNewItem(prev => ({ ...prev, [name]: value === '' ? '' : parseFloat(value) || 0 }));
    };

    const handleAddOrUpdateItem = async () => {
        if (!db || !userId || !newItem.name.trim()) {
            setMessage("Name cannot be empty.");
            setShowMessageModal(true);
            return;
        }

        const updatedOptions = { ...profile.jobWorkflowOptions };
        const currentList = updatedOptions[currentOptionType];

        const itemToAdd = {
            name: newItem.name.trim(),
            cost: parseFloat(newItem.cost) || 0,
            discountType: newItem.discountType || 'None',
            discountValue: parseFloat(newItem.discountValue) || 0,
            quantity: parseInt(newItem.quantity) || 1,
        };

        if (editingIndex !== null) {
            // Update existing item
            currentList[editingIndex] = itemToAdd;
            setMessage("Item updated successfully!");
        } else {
            // Add new item
            currentList.push(itemToAdd);
            setMessage("Item added successfully!");
        }

        try {
            // Corrected Firestore path: artifacts/{appId}/users/{userId}/profile/companyData
            await setDoc(doc(db, 'artifacts', appId, 'users', userId, 'profile', 'companyData'), { jobWorkflowOptions: updatedOptions }, { merge: true });
            setProfile(prev => ({ ...prev, jobWorkflowOptions: updatedOptions }));
            setNewItem({ name: '', cost: '', discountType: 'None', discountValue: '', quantity: '' }); // Clear form
            setEditingIndex(null); // Exit editing mode
            setShowMessageModal(true);
        } catch (error) {
            console.error("Error saving workflow item:", error);
            setMessage("Error saving workflow item.");
            setShowMessageModal(true);
        }
    };

    const handleEditItem = (index) => {
        const itemToEdit = profile.jobWorkflowOptions[currentOptionType][index];
        setNewItem({
            name: itemToEdit.name,
            cost: itemToEdit.cost,
            discountType: itemToEdit.discountType,
            discountValue: itemToEdit.discountValue,
            quantity: itemToEdit.quantity,
        });
        setEditingIndex(index);
    };

    const handleDeleteItem = async (index) => {
        if (!db || !userId) return;

        const updatedOptions = { ...profile.jobWorkflowOptions };
        updatedOptions[currentOptionType].splice(index, 1);

        try {
            // Corrected Firestore path: artifacts/{appId}/users/{userId}/profile/companyData
            await setDoc(doc(db, 'artifacts', appId, 'users', userId, 'profile', 'companyData'), { jobWorkflowOptions: updatedOptions }, { merge: true });
            setProfile(prev => ({ ...prev, jobWorkflowOptions: updatedOptions }));
            setMessage("Item deleted successfully!");
            setShowMessageModal(true);
        } catch (error) {
            console.error("Error deleting workflow item:", error);
            setMessage("Error deleting workflow item.");
            setShowMessageModal(true);
        }
    };

    const handleCancelEdit = () => {
        setNewItem({ name: '', cost: '', discountType: 'None', discountValue: '', quantity: '' });
        setEditingIndex(null);
    };

    const currentListToDisplay = profile.jobWorkflowOptions[currentOptionType] || [];

    return (
        <div className="p-4 sm:p-6 bg-gray-50 min-h-screen">
            <h2 className="text-2xl sm:text-3xl font-bold text-gray-800 mb-6">Edit Job Workflow Options</h2>

            <div className="bg-white p-4 sm:p-6 rounded-lg shadow-md mb-6">
                <div className="flex flex-wrap gap-2 mb-4">
                    <button
                        type="button"
                        onClick={() => setCurrentOptionType('glassTypes')}
                        className={`py-2 px-4 rounded-lg font-semibold transition duration-300 ${currentOptionType === 'glassTypes' ? 'bg-blue-600 text-white shadow-md' : 'bg-gray-200 text-gray-800 hover:bg-gray-300'}`}
                    >
                        Glass Types
                    </button>
                    <button
                        type="button"
                        onClick={() => setCurrentOptionType('damageTypes')}
                        className={`py-2 px-4 rounded-lg font-semibold transition duration-300 ${currentOptionType === 'damageTypes' ? 'bg-blue-600 text-white shadow-md' : 'bg-gray-200 text-gray-800 hover:bg-gray-300'}`}
                    >
                        Damage Types
                    </button>
                    <button
                        type="button"
                        onClick={() => setCurrentOptionType('repairReplacementOptions')}
                        className={`py-2 px-4 rounded-lg font-semibold transition duration-300 ${currentOptionType === 'repairReplacementOptions' ? 'bg-blue-600 text-white shadow-md' : 'bg-gray-200 text-gray-800 hover:bg-gray-300'}`}
                    >
                        Repair/Replacement Options
                    </button>
                </div>

                <h3 className="text-xl font-semibold text-gray-800 mb-4">Add/Edit {currentOptionType.replace(/([A-Z])/g, ' $1').toLowerCase()}</h3>
                <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4 mb-4">
                    <div>
                        <label htmlFor="itemName" className="block text-sm font-medium text-gray-700 mb-1">Name</label>
                        <input
                            type="text"
                            id="itemName"
                            name="name"
                            value={newItem.name}
                            onChange={handleNewItemChange}
                            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                            placeholder="Item Name"
                            required
                        />
                    </div>
                    <div>
                        <label htmlFor="itemCost" className="block text-sm font-medium text-gray-700 mb-1">Cost ($)</label>
                        <input
                            type="number"
                            id="itemCost"
                            name="cost"
                            value={newItem.cost}
                            onChange={handleNewItemNumberChange}
                            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                            min="0"
                            step="0.01"
                        />
                    </div>
                    <div>
                        <label htmlFor="itemQuantity" className="block text-sm font-medium text-gray-700 mb-1">Quantity</label>
                        <input
                            type="number"
                            id="itemQuantity"
                            name="quantity"
                            value={newItem.quantity}
                            onChange={handleNewItemNumberChange}
                            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                            min="1"
                            step="1"
                        />
                    </div>
                    <div>
                        <label htmlFor="itemDiscountType" className="block text-sm font-medium text-gray-700 mb-1">Discount Type</label>
                        <select
                            id="itemDiscountType"
                            name="discountType"
                            value={newItem.discountType}
                            onChange={handleNewItemChange}
                            className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                        >
                            <option value="None">None</option>
                            <option value="Percentage">Percentage (%)</option>
                            <option value="Flat Rate">Flat Rate ($)</option>
                        </select>
                    </div>
                    {newItem.discountType !== 'None' && (
                        <div>
                            <label htmlFor="itemDiscountValue" className="block text-sm font-medium text-gray-700 mb-1">Discount Value</label>
                            <input
                                type="number"
                                id="itemDiscountValue"
                                name="discountValue"
                                value={newItem.discountValue}
                                onChange={handleNewItemNumberChange}
                                className="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                                min="0"
                                step="0.01"
                            />
                        </div>
                    )}
                </div>
                <div className="flex flex-col sm:flex-row space-y-2 sm:space-y-0 sm:space-x-4">
                    <button
                        type="button"
                        onClick={handleAddOrUpdateItem}
                        className="bg-green-600 hover:bg-green-700 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md flex-1"
                    >
                        {editingIndex !== null ? 'Update Item' : 'Add Item'}
                    </button>
                    {editingIndex !== null && (
                        <button
                            type="button"
                            onClick={handleCancelEdit}
                            className="bg-gray-500 hover:bg-gray-600 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md flex-1"
                        >
                            Cancel Edit
                        </button>
                    )}
                </div>
            </div>

            <div className="bg-white p-4 sm:p-6 rounded-lg shadow-md mb-6">
                <h3 className="text-xl font-semibold text-gray-800 mb-4">{currentOptionType.replace(/([A-Z])/g, ' $1').toLowerCase()} List</h3>
                {currentListToDisplay.length === 0 ? (
                    <p className="text-gray-600">No items added yet for this category.</p>
                ) : (
                    <div className="overflow-x-auto">
                        <table className="min-w-full divide-y divide-gray-200">
                            <thead className="bg-gray-50">
                                <tr>
                                    <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Name</th>
                                    <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Cost</th>
                                    <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Qty</th>
                                    <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Discount</th>
                                    <th className="px-3 py-2 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Actions</th>
                                </tr>
                            </thead>
                            <tbody className="bg-white divide-y divide-gray-200">
                                {currentListToDisplay.map((item, index) => (
                                    <tr key={index}>
                                        <td className="px-3 py-2 whitespace-nowrap text-sm font-medium text-gray-900">{item.name}</td>
                                        <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-700">{formatCurrency(item.cost)}</td>
                                        <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-700">{item.quantity}</td>
                                        <td className="px-3 py-2 whitespace-nowrap text-sm text-gray-700">
                                            {item.discountType === 'Percentage' ? `${item.discountValue}%` : formatCurrency(item.discountValue)} ({item.discountType})
                                        </td>
                                        <td className="px-3 py-2 whitespace-nowrap text-sm font-medium">
                                            <button
                                                type="button"
                                                onClick={() => handleEditItem(index)}
                                                className="text-blue-600 hover:text-blue-900 mr-2"
                                            >
                                                Edit
                                            </button>
                                            <button
                                                type="button"
                                                onClick={() => handleDeleteItem(index)}
                                                className="text-red-600 hover:text-red-900"
                                            >
                                                Delete
                                            </button>
                                        </td>
                                    </tr>
                                ))}
                            </tbody>
                        </table>
                    </div>
                )}
            </div>

            <div className="flex justify-end mt-6">
                <button
                    type="button"
                    onClick={() => navigate('settings')}
                    className="bg-gray-500 hover:bg-gray-600 text-white font-bold py-2 px-4 rounded-lg transition duration-300 shadow-md w-full sm:w-auto"
                >
                    Back to Settings
                </button>
            </div>
            {showMessageModal && (
                <MessageModal
                    message={message}
                    onClose={() => setShowMessageModal(false)}
                />
            )}
        </div>
    );
};

// Main App Component
const App = () => {
    const [currentPage, setCurrentPage] = useState('dashboard');
    const [pageProps, setPageProps] = useState({});

    const navigate = (page, props = {}) => {
        setCurrentPage(page);
        setPageProps(props);
    };

    const renderPage = () => {
        switch (currentPage) {
            case 'dashboard':
                return <Dashboard navigate={navigate} />;
            case 'customers':
                return <Customers navigate={navigate} />;
            case 'addCustomer':
                return <AddCustomer navigate={navigate} {...pageProps} />;
            case 'jobDetail':
                return <JobDetail navigate={navigate} {...pageProps} />;
            case 'settings':
                return <Settings navigate={navigate} />;
            case 'jobWorkflowSettings':
                return <JobWorkflowSettings navigate={navigate} />;
            default:
                return <Dashboard navigate={navigate} />;
        }
    };

    return (
        <AppProvider>
            <div className="min-h-screen bg-gray-100 flex flex-col">
                {/* Header */}
                <header className="bg-blue-700 text-white p-4 shadow-md">
                    <div className="container mx-auto flex flex-col sm:flex-row justify-between items-center">
                        <h1 className="text-2xl font-bold mb-2 sm:mb-0">ProSto Auto Glass</h1>
                        {/* User ID display - crucial for multi-user apps */}
                        <AppContext.Consumer>
                            {({ userId }) => userId && (
                                <p className="text-sm text-blue-200">User ID: {userId}</p>
                            )}
                        </AppContext.Consumer>
                        {/* Settings button moved to header for better mobile access */}
                        <button
                            onClick={() => navigate('settings')}
                            className="rounded-full w-10 h-10 flex items-center justify-center bg-blue-600 text-white shadow-lg
                                      hover:bg-blue-700 transition duration-300 focus:outline-none focus:ring-2 focus:ring-blue-400 mt-2 sm:mt-0"
                            title="Settings"
                        >
                            <svg xmlns="http://www.w3.org/2000/svg" className="h-6 w-6" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                <circle cx="12" cy="12" r="3"></circle>
                                <path d="M19.4 15a1.65 1.65 0 00.33 1.82l.06.06a2 2 0 010 2.83 2 2 0 01-2.83 0l-.06-.06a1.65 1.65 0 00-1.82-.33 1.65 1.65 0 00-1 1.51V21a2 2 0 01-2 2 2 2 0 01-2-2v-.09A1.65 1.65 0 009 19.4a1.65 1.65 0 00-1.82.33l-.06.06a2 2 0 01-2.83 0 2 2 0 010-2.83l.06-.06a1.65 1.65 0 00-.33-1.82 1.65 1.65 0 00-1.51-1H3a2 2 0 01-2-2 2 2 0 012-2h.09A1.65 1.65 0 004.6 9a1.65 1.65 0 00-.33-1.82l-.06-.06a2 2 0 010-2.83 2 2 0 012.83 0l.06.06a1.65 1.65 0 001.82-.33V3a2 2 0 012-2 2 2 0 012 2v.09A1.65 1.65 0 0015 4.6a1.65 1.65 0 001.82-.33l.06-.06a2 2 0 012.83 0 2 2 0 010 2.83l-.06.06a1.65 1.65 0 00.33 1.82 1.65 1.65 0 001.51 1H21a2 2 0 012 2 2 2 0 01-2 2h-.09a1.65 1.65 0 00-1.51 1z"></path>
                            </svg>
                        </button>
                    </div>
                </header>

                {/* Navigation */}
                <nav className="bg-blue-600 text-white shadow-md py-2 px-4">
                    <ul className="flex flex-wrap justify-center sm:justify-start gap-x-4 gap-y-2 text-sm sm:text-base">
                        <li>
                            <button
                                onClick={() => navigate('dashboard')}
                                className={`py-2 px-3 rounded-lg transition duration-300 ${currentPage === 'dashboard' ? 'bg-blue-800 font-bold' : 'hover:bg-blue-500'}`}
                            >
                                Dashboard
                            </button>
                        </li>
                        <li>
                            <button
                                onClick={() => navigate('customers')}
                                className={`py-2 px-3 rounded-lg transition duration-300 ${currentPage === 'customers' || currentPage === 'addCustomer' ? 'bg-blue-800 font-bold' : 'hover:bg-blue-500'}`}
                            >
                                Customers
                            </button>
                        </li>
                        <li>
                            <button
                                onClick={() => navigate('jobDetail')}
                                className={`py-2 px-3 rounded-lg transition duration-300 ${currentPage === 'jobDetail' ? 'bg-blue-800 font-bold' : 'hover:bg-blue-500'}`}
                            >
                                Jobs
                            </button>
                        </li>
                        {/* Settings button is now in the header, so removed from here */}
                    </ul>
                </nav>

                {/* Main Content */}
                <main className="flex-grow container mx-auto px-2 py-4 sm:px-4 sm:py-6">
                    {renderPage()}
                </main>
            </div>
        </AppProvider>
    );
};

export default App;
