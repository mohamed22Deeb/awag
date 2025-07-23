

<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>شجرة عائلة أعواج التفاعلية</title>
    <!-- Tailwind CSS CDN for easy styling -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Firebase SDKs -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, addDoc, onSnapshot, query, deleteDoc, doc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Global variables provided by the Canvas environment
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        let app;
        let db;
        let auth;
        let currentUserId = null;
        let familyMembersRef = null;
        let familyData = []; // Array to hold all family members
        let activeExpandedBranchIds = new Set(); // To track all currently expanded branches
        let isAdmin = false; // Flag to control admin features
        const ADMIN_PASSWORD = "082"; // The admin password

        // Initialize Firebase and set up authentication
        const initFirebase = async () => {
            console.log("Initializing Firebase...");
            try {
                app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);
                console.log("Firebase app and services initialized.");

                // Listen for authentication state changes
                onAuthStateChanged(auth, async (user) => {
                    if (user) {
                        currentUserId = user.uid;
                        document.getElementById('user-id-display').textContent = `معرف المستخدم: ${currentUserId}`;
                        console.log("User authenticated:", currentUserId);
                        familyMembersRef = collection(db, `artifacts/${appId}/users/${currentUserId}/familyMembers`);
                        setupRealtimeListener();
                    } else {
                        console.log("No user authenticated. Attempting sign-in...");
                        if (initialAuthToken) {
                            try {
                                await signInWithCustomToken(auth, initialAuthToken);
                                console.log("Signed in with custom token.");
                            } catch (error) {
                                console.error("Error signing in with custom token, falling back to anonymous:", error);
                                await signInAnonymously(auth);
                                console.log("Signed in anonymously after custom token failure.");
                            }
                        } else {
                            await signInAnonymously(auth);
                            console.log("Signed in anonymously.");
                        }
                    }
                });
            } catch (error) {
                console.error("Critical Error initializing Firebase:", error);
                document.getElementById('family-tree-render-area').innerHTML = `<p class="text-red-500">حدث خطأ فادح في تحميل التطبيق: ${error.message}. يرجى التحقق من إعدادات Firebase.</p>`;
                document.getElementById('loading-message').classList.add('hidden');
            }
        };

        // Set up real-time listener for family data
        const setupRealtimeListener = () => {
            if (!familyMembersRef) {
                console.warn("Family members collection reference is not available. Cannot set up listener.");
                document.getElementById('loading-message').classList.add('hidden');
                return;
            }

            console.log("Setting up real-time listener for:", familyMembersRef.path);
            const q = query(familyMembersRef);

            onSnapshot(q, (snapshot) => {
                const members = [];
                snapshot.forEach((doc) => {
                    members.push({ id: doc.id, ...doc.data() });
                });
                familyData = members.sort((a, b) => a.name.localeCompare(b.name));
                console.log(`Received ${familyData.length} family members from Firestore.`);
                renderFamilyTree();
                populateParentSelects();
                document.getElementById('loading-message').classList.add('hidden');
                updateAdminButtonsVisibility(); // Update visibility after rendering
            }, (error) => {
                console.error("Error listening to Firestore updates:", error);
                document.getElementById('family-tree-render-area').innerHTML = `<p class="text-red-500">حدث خطأ في جلب بيانات العائلة: ${error.message}</p>`;
                document.getElementById('loading-message').classList.add('hidden');
            });
        };

        // Recursive function to render a single member and their children
        function renderMemberCard(member) {
            const photoSrc = member.photoUrl && member.photoUrl.startsWith('http') ? member.photoUrl : `https://placehold.co/80x80/CBD5E0/FFFFFF?text=${member.name.split(' ')[0]}`;
            
            const parentNames = [];
            if (member.fatherId) {
                const father = familyData.find(f => f.id === member.fatherId);
                if (father) parentNames.push(father.name);
            }
            if (member.motherId) {
                const mother = familyData.find(f => f.id === member.motherId);
                if (mother) parentNames.push(mother.name);
            }
            const parentsDisplay = parentNames.length > 0 ? `<small class="text-gray-500 text-xs mt-1">الوالدان: ${parentNames.join(' و ')}</small>` : '';

            let roleClass = '';
            if (member.role === 'grandparent') roleClass = 'grand-parent';
            else if (member.role === 'parent') roleClass = 'parent';
            else if (member.role === 'child') roleClass = 'child';

            const directChildren = familyData.filter(child => child.fatherId === member.id || child.motherId === member.id);
            const hasChildren = directChildren.length > 0;
            const expandedClass = activeExpandedBranchIds.has(member.id) ? 'expanded' : '';

            // Conditionally add delete button HTML based on isAdmin flag
            const deleteButtonHtml = `
                <button class="delete-button absolute top-2 left-2 bg-red-500 text-white rounded-full w-6 h-6 flex items-center justify-center text-xs opacity-80 hover:opacity-100 transition-opacity admin-only-feature hidden" data-member-id="${member.id}">
                    &times;
                </button>
            `;

            let html = `
                <div class="family-node">
                    <div id="${member.id}" class="person ${roleClass} ${hasChildren ? 'has-children' : ''} ${expandedClass} rounded-xl p-5 shadow-lg flex flex-col items-center justify-center transform transition-all duration-300 hover:-translate-y-2 hover:shadow-xl cursor-pointer relative">
                        ${deleteButtonHtml} <!-- Delete button added here -->
                        <div class="photo-placeholder w-20 h-20 rounded-full mb-3 border-4 border-white flex items-center justify-center overflow-hidden">
                            <img src="${photoSrc}" alt="صورة ${member.name}" class="w-full h-full object-cover" onerror="this.onerror=null;this.src='https://placehold.co/80x80/CBD5E0/FFFFFF?text=${member.name.split(' ')[0]}';">
                        </div>
                        <strong class="text-xl font-semibold text-gray-800 mb-1">${member.name}</strong>
                        <small class="text-gray-600 text-sm leading-tight">${member.birthDate ? `مواليد: ${member.birthDate}` : ''}${member.deathDate ? ` - وفاة: ${member.deathDate}` : ''}</small>
                        ${parentsDisplay}
                        ${hasChildren ? '<div class="expand-icon"></div>' : ''}
                    </div>
            `;

            if (hasChildren) {
                const hiddenClass = activeExpandedBranchIds.has(member.id) ? '' : 'hidden';
                html += `<div id="children-of-${member.id}" class="children-container ${hiddenClass}">`;
                directChildren.forEach(child => {
                    html += renderMemberCard(child); // Recursively render children
                });
                html += `</div>`;
            }

            html += `</div>`; // Close family-node
            return html;
        }

        // Main render function
        const renderFamilyTree = () => {
            const treeArea = document.getElementById('family-tree-render-area');
            treeArea.innerHTML = '';
            
            if (familyData.length === 0) {
                treeArea.innerHTML = `<p class="text-gray-600 text-lg mt-8">لا يوجد أفراد في شجرة العائلة بعد. <br> استخدم زر "إضافة فرد جديد" لإضافة أول فرد!</p>`;
                return;
            }

            // Find top-level members (those without any parents assigned)
            const topLevelMembers = familyData.filter(m => !m.fatherId && !m.motherId);

            if (topLevelMembers.length === 0 && familyData.length > 0) {
                // Fallback: If no explicit top-level members, render all members as a flat list initially
                console.warn("No top-level members found. Rendering all members as a flat list.");
                familyData.forEach(member => {
                    treeArea.innerHTML += renderMemberCard(member);
                });
            } else {
                topLevelMembers.forEach(member => {
                    treeArea.innerHTML += renderMemberCard(member);
                });
            }

            // Add click listeners to all person cards using event delegation
            treeArea.onclick = (event) => { // Use onclick directly on container for delegation
                const target = event.target;

                // Handle delete button click
                if (target.classList.contains('delete-button') && isAdmin) {
                    const memberIdToDelete = target.dataset.memberId;
                    showConfirmModal("هل أنت متأكد أنك تريد حذف هذا الفرد؟", () => deleteMember(memberIdToDelete));
                    return; // Prevent expand/collapse when delete button is clicked
                }

                // Handle person card click (expand/collapse)
                const personDiv = target.closest('.person');
                if (personDiv && !target.classList.contains('delete-button')) { // Ensure not clicking delete button
                    handlePersonClick(personDiv.id);
                }
            };

            // Scroll to the first expanded element to ensure visibility on re-render
            if (activeExpandedBranchIds.size > 0) {
                const firstExpandedId = activeExpandedBranchIds.values().next().value;
                const firstExpandedEl = document.getElementById(firstExpandedId);
                if (firstExpandedEl) {
                    firstExpandedEl.scrollIntoView({ behavior: 'smooth', block: 'center' });
                }
            }
        };

        // Function to update visibility of admin-only buttons
        const updateAdminButtonsVisibility = () => {
            document.getElementById('add-member-btn').classList.toggle('hidden', !isAdmin);
            document.querySelectorAll('.delete-button.admin-only-feature').forEach(btn => {
                btn.classList.toggle('hidden', !isAdmin);
            });
        };

        // Handle person click to expand/collapse children
        function handlePersonClick(personId) {
            console.log("Clicked person for expand/collapse:", personId);
            const clickedPersonEl = document.getElementById(personId);
            if (!clickedPersonEl) {
                console.error("Clicked person element not found:", personId);
                return;
            }

            const childrenContainer = document.getElementById(`children-of-${personId}`);
            const hasChildren = clickedPersonEl.classList.contains('has-children');

            // If the clicked person has no children, just toggle highlight and return
            if (!hasChildren) {
                clickedPersonEl.classList.toggle('highlight-parent');
                if (!clickedPersonEl.classList.contains('highlight-parent')) {
                    activeExpandedBranchIds.delete(personId);
                }
                console.log("Person has no children. Toggling highlight only. activeExpandedBranchIds:", activeExpandedBranchIds);
                return;
            }

            // Toggle expansion state
            if (activeExpandedBranchIds.has(personId)) {
                // Collapse this branch
                clickedPersonEl.classList.remove('expanded', 'highlight-parent');
                if (childrenContainer) childrenContainer.classList.add('hidden');
                activeExpandedBranchIds.delete(personId);
                console.log("Collapsed branch:", personId, "activeExpandedBranchIds:", activeExpandedBranchIds);
            } else {
                // Expand this branch
                clickedPersonEl.classList.add('expanded', 'highlight-parent');
                if (childrenContainer) childrenContainer.classList.remove('hidden');
                activeExpandedBranchIds.add(personId);
                console.log("Expanded branch:", personId, "activeExpandedBranchIds:", activeExpandedBranchIds);

                clickedPersonEl.scrollIntoView({ behavior: 'smooth', block: 'center' });
            }
        }

        // Function to delete a family member
        const deleteMember = async (memberId) => {
            if (!db || !familyMembersRef) {
                showAlert("خطأ: قاعدة البيانات غير مهيأة.");
                return;
            }
            try {
                await deleteDoc(doc(db, familyMembersRef.path, memberId));
                showAlert("تم حذف الفرد بنجاح.");
                // Data will re-render automatically due to onSnapshot listener
            } catch (error) {
                console.error("Error deleting member:", error);
                showAlert("حدث خطأ أثناء حذف الفرد: " + error.message);
            }
        };


        // Populate father and mother selection dropdowns in the add member form
        const populateParentSelects = () => {
            const fatherSelect = document.getElementById('new-member-father');
            const motherSelect = document.getElementById('new-member-mother');

            fatherSelect.innerHTML = '<option value="">اختر الأب (اختياري)</option>';
            motherSelect.innerHTML = '<option value="">اختر الأم (اختياري)</option>';

            familyData.forEach(member => {
                const option = document.createElement('option');
                option.value = member.id;
                option.textContent = member.name;

                if (member.gender === 'male' || member.role === 'grandparent' || member.role === 'parent') {
                    fatherSelect.appendChild(option.cloneNode(true));
                }
                if (member.gender === 'female' || member.role === 'grandparent' || member.role === 'parent') {
                    motherSelect.appendChild(option.cloneNode(true));
                }
            });
        };

        // Show/Hide Add New Member Modal
        const toggleAddMemberModal = (show) => {
            const modal = document.getElementById('add-member-modal');
            if (show) {
                modal.classList.remove('hidden');
                populateParentSelects(); // Populate parents when modal opens
            } else {
                modal.classList.add('hidden');
                document.getElementById('add-member-form').reset(); // Clear form
            }
        };

        // Handle adding a new family member
        const addNewMember = async (event) => {
            event.preventDefault(); // Prevent default form submission

            const form = event.target;
            const name = form['new-member-name'].value;
            const role = form['new-member-role'].value;
            const gender = form['new-member-gender'].value;
            const birthDate = form['new-member-birthdate'].value;
            const deathDate = form['new-member-deathdate'].value;
            const photoUrl = form['new-member-photo'].value;
            const fatherId = form['new-member-father'].value;
            const motherId = form['new-member-mother'].value;

            if (!name || !role || !gender) {
                showAlert("الرجاء إدخال الاسم، الدور، والجنس.");
                return;
            }

            const newMember = {
                name: name,
                role: role,
                gender: gender,
                birthDate: birthDate,
                deathDate: deathDate,
                photoUrl: photoUrl,
                fatherId: fatherId || null,
                motherId: motherId || null
            };

            try {
                await addDoc(familyMembersRef, newMember);
                toggleAddMemberModal(false);
            } catch (error) {
                console.error("Error adding new member:", error);
                showAlert("حدث خطأ أثناء إضافة الفرد: " + error.message);
            }
        };

        // Show/Hide Awaj Info Modal
        const toggleAwajInfoModal = (show) => {
            const modal = document.getElementById('awaj-info-modal');
            if (show) {
                modal.classList.remove('hidden');
            } else {
                modal.classList.add('hidden');
            }
        };

        // Custom Confirmation Modal (replaces window.confirm)
        function showConfirmModal(message, onConfirm) {
            const confirmModal = document.getElementById('custom-confirm-modal');
            document.getElementById('custom-confirm-message').textContent = message;
            confirmModal.classList.remove('hidden');

            document.getElementById('custom-confirm-ok-btn').onclick = () => {
                confirmModal.classList.add('hidden');
                onConfirm();
            };
            document.getElementById('custom-confirm-cancel-btn').onclick = () => {
                confirmModal.classList.add('hidden');
            };
        }

        // Show/Hide Password Modal
        const togglePasswordModal = (show) => {
            const modal = document.getElementById('password-modal');
            if (show) {
                modal.classList.remove('hidden');
                document.getElementById('admin-password-input').value = ''; // Clear input on show
                document.getElementById('admin-password-input').focus(); // Focus input
            } else {
                modal.classList.add('hidden');
            }
        };

        // Event Listeners
        document.addEventListener('DOMContentLoaded', () => {
            initFirebase();

            document.getElementById('add-member-btn').addEventListener('click', () => toggleAddMemberModal(true));
            document.getElementById('close-modal-btn').addEventListener('click', () => toggleAddMemberModal(false));
            document.getElementById('add-member-form').addEventListener('submit', addNewMember);

            // Event listeners for Awaj Info Modal
            document.getElementById('awaj-info-btn').addEventListener('click', () => toggleAwajInfoModal(true));
            document.getElementById('close-awaj-info-modal-btn').addEventListener('click', () => toggleAwajInfoModal(false));

            // Admin Mode Toggle
            document.getElementById('admin-mode-toggle').addEventListener('change', (event) => {
                if (event.target.checked) {
                    // If trying to enable admin mode, show password modal
                    togglePasswordModal(true);
                } else {
                    // If trying to disable admin mode, directly disable
                    isAdmin = false;
                    updateAdminButtonsVisibility();
                    renderFamilyTree(); // Re-render to hide delete buttons
                }
            });

            // Password Modal buttons
            document.getElementById('password-submit-btn').addEventListener('click', () => {
                const enteredPassword = document.getElementById('admin-password-input').value;
                if (enteredPassword === ADMIN_PASSWORD) {
                    isAdmin = true;
                    togglePasswordModal(false);
                    updateAdminButtonsVisibility();
                    renderFamilyTree(); // Re-render to show delete buttons
                } else {
                    showAlert("كلمة المرور خاطئة. يرجى المحاولة مرة أخرى.");
                    document.getElementById('admin-password-input').value = ''; // Clear input
                }
            });

            document.getElementById('password-cancel-btn').addEventListener('click', () => {
                togglePasswordModal(false);
                document.getElementById('admin-mode-toggle').checked = false; // Uncheck the toggle if cancelled
                isAdmin = false; // Ensure admin mode is off
                updateAdminButtonsVisibility();
                renderFamilyTree(); // Re-render to hide delete buttons
            });
        });

        // Simple modal for alerts (instead of window.alert)
        function showAlert(message) {
            const alertModal = document.getElementById('custom-alert-modal');
            document.getElementById('custom-alert-message').textContent = message;
            alertModal.classList.remove('hidden');
            document.getElementById('custom-alert-ok-btn').onclick = () => alertModal.classList.add('hidden');
        }

        // Override default alert for demonstration purposes (optional, but good practice)
        window.alert = showAlert;

    </script>
    <style>
        /* Base styles */
        body {
            font-family: 'Inter', sans-serif;
            background-color: #F5F8FA; /* Very light blue-gray background */
            display: flex;
            flex-direction: column;
            justify-content: flex-start;
            align-items: center;
            min-height: 100vh;
            margin: 0;
            padding: 20px;
            direction: rtl; /* Right-to-left direction for Arabic */
            overflow-x: auto; /* Allow horizontal scrolling for large trees */
        }

        /* Main container for the entire application */
        .w-full.max-w-7xl {
            width: 100%;
            max-width: 1400px;
        }

        /* Header and Admin Controls */
        .bg-white.p-6.rounded-xl.shadow-md {
            background-color: #FFFFFF;
            padding: 24px;
            border-radius: 16px;
            box-shadow: 0 10px 25px rgba(0, 0, 0, 0.08); /* Softer shadow */
            width: 100%;
            max-width: 1200px;
            margin-bottom: 32px;
            display: flex;
            flex-direction: column; /* Changed to column to stack title and buttons */
            align-items: center; /* Center items */
            gap: 16px;
        }

        @media (min-width: 768px) {
            .bg-white.p-6.rounded-xl.shadow-md {
                flex-direction: row; /* Revert to row on larger screens */
                justify-content: space-between;
            }
        }

        /* Container for Title and About Button */
        .title-and-about-btn {
            display: flex;
            flex-direction: column;
            align-items: center;
            gap: 8px; /* Space between title and about button */
        }

        h1 {
            color: #334155; /* Dark slate for heading */
            margin-bottom: 0; /* No margin as gap handles spacing */
            font-size: 2.8em;
            font-weight: 700;
            text-shadow: 1px 1px 2px rgba(0,0,0,0.03); /* Softer text shadow */
            text-align: center;
        }

        /* About Button (New Style) */
        #awaj-info-btn {
            background-color: #E2E8F0; /* Light gray background */
            color: #475569; /* Darker text */
            font-weight: 500;
            padding: 6px 15px; /* Smaller padding */
            border-radius: 8px; /* Slightly less rounded */
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05); /* Very subtle shadow */
            transition: all 0.2s ease-in-out;
            border: 1px solid #CBD5E0; /* Light border */
            cursor: pointer;
            font-size: 0.9em; /* Smaller font size */
        }

        #awaj-info-btn:hover {
            background-color: #CBD5E0; /* Slightly darker gray on hover */
            transform: translateY(-1px) scale(1.01);
            box-shadow: 0 3px 6px rgba(0, 0, 0, 0.1);
        }

        /* Admin Controls (Add Member Button, Admin Toggle, User ID Display) */
        .admin-controls {
            display: flex;
            flex-direction: column; /* Stack vertically on mobile */
            align-items: center;
            gap: 10px; /* Space between elements */
        }

        @media (min-width: 768px) {
            .admin-controls {
                flex-direction: row; /* Align horizontally on larger screens */
                gap: 16px; /* Space between elements */
            }
        }

        /* Admin Mode Toggle Checkbox */
        #admin-mode-toggle {
            /* Basic styling for the checkbox itself */
            appearance: none; /* Remove default browser styling */
            -webkit-appearance: none;
            -moz-appearance: none;
            width: 20px;
            height: 20px;
            border: 2px solid #CBD5E0;
            border-radius: 4px;
            background-color: #F8F9FA;
            cursor: pointer;
            position: relative;
            outline: none;
            transition: all 0.2s ease;
        }

        #admin-mode-toggle:checked {
            background-color: #4299E1; /* Blue when checked */
            border-color: #4299E1;
        }

        #admin-mode-toggle:checked::before {
            content: '✔'; /* Checkmark */
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            font-size: 14px;
            color: white;
        }

        /* Add Member Button */
        #add-member-btn {
            background-color: #4299E1; /* Tailwind blue-500 */
            color: white;
            font-weight: 600;
            padding: 12px 28px;
            border-radius: 10px;
            box-shadow: 0 4px 8px rgba(66, 153, 225, 0.2);
            transition: all 0.3s ease-in-out;
            border: none;
            cursor: pointer;
            flex-shrink: 0;
        }

        #add-member-btn:hover {
            background-color: #3182CE;
            transform: translateY(-3px) scale(1.02);
            box-shadow: 0 6px 12px rgba(66, 153, 225, 0.3);
        }

        /* User ID Display */
        #user-id-display {
            color: #64748B;
            font-size: 0.9em;
            text-align: center;
            word-break: break-all;
        }

        /* Loading Message */
        #loading-message {
            color: #64748B;
            font-size: 1.2em;
            margin-top: 50px;
        }

        /* Family Tree Container */
        .family-tree-container {
            background-color: #FFFFFF;
            padding: 40px;
            border-radius: 20px;
            box-shadow: 0 15px 40px rgba(0, 0, 0, 0.1);
            width: 100%;
            max-width: 1400px;
            text-align: center;
            position: relative;
            min-height: 600px;
            overflow-x: auto;
            display: flex;
            flex-direction: column;
            align-items: center;
            gap: 40px;
        }

        /* Individual Person Card */
        .person {
            background: linear-gradient(135deg, #FFFFFF 0%, #F8F9FA 100%);
            border: 1px solid #E2E8F0;
            border-radius: 15px;
            padding: 25px 30px;
            box-shadow: 0 8px 16px rgba(0, 0, 0, 0.08);
            min-width: 220px;
            max-width: 280px;
            text-align: center;
            transition: all 0.3s ease-in-out;
            position: relative;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            flex-grow: 0;
            flex-shrink: 0;
            cursor: pointer;
            margin-bottom: 0;
        }

        .person:hover {
            transform: translateY(-10px) scale(1.03);
            box-shadow: 0 15px 30px rgba(0, 0, 0, 0.15);
            background: linear-gradient(135deg, #F8F9FA 0%, #EFEFF1 100%);
        }

        .person strong {
            margin: 10px 0 5px;
            color: #334155;
            font-size: 1.4em;
            font-weight: 700;
        }

        .person small {
            color: #64748B;
            font-size: 0.95em;
            line-height: 1.5;
        }

        /* Photo Placeholder */
        .photo-placeholder {
            width: 90px;
            height: 90px;
            background-color: #CBD5E0;
            border-radius: 50%;
            margin-bottom: 15px;
            border: 4px solid #F0F4F8;
            display: flex;
            justify-content: center;
            align-items: center;
            color: #FFFFFF;
            font-size: 1em;
            font-weight: bold;
            overflow: hidden;
            flex-shrink: 0;
        }

        .photo-placeholder img {
            width: 100%;
            height: 100%;
            object-fit: cover;
        }

        /* Specific styling for roles with refined colors */
        .grand-parent {
            background: linear-gradient(135deg, #FDF8E1 0%, #FBEEDC 100%);
            border-color: #D4AF37;
        }
        .grand-parent strong { color: #B8860B; }
        .grand-parent .photo-placeholder { background-color: #D4AF37; border-color: #FDF8E1; }

        .parent {
            background: linear-gradient(135deg, #E6F7F3 0%, #D0EDE4 100%);
            border-color: #4CAF50;
        }
        .parent strong { color: #2E8B57; }
        .parent .photo-placeholder { background-color: #4CAF50; border-color: #E6F7F3; }

        .child {
            background: linear-gradient(135deg, #EAF3FF 0%, #D9E8FF 100%);
            border-color: #6495ED;
        }
        .child strong { color: #4169E1; }
        .child .photo-placeholder { background-color: #6495ED; border-color: #EAF3FF; }

        /* Highlight styles for interactivity */
        .highlight-parent {
            box-shadow: 0 0 0 5px #4299E1, 0 15px 30px rgba(0, 0, 0, 0.2) !important;
        }

        /* Expand/Collapse Icon */
        .expand-icon {
            position: absolute;
            bottom: 5px;
            right: 5px;
            width: 24px;
            height: 24px;
            background-color: #64748B;
            border-radius: 50%;
            display: flex;
            align-items: center;
            justify-content: center;
            color: white;
            font-size: 1.4em;
            line-height: 1;
            transition: transform 0.3s ease;
        }
        .expand-icon::before {
            content: '+';
        }
        .person.expanded .expand-icon::before {
            content: '-';
            transform: rotate(0deg);
        }
        .person.expanded .expand-icon {
            transform: rotate(180deg);
        }


        /* Family Node - wrapper for person and its children container */
        .family-node {
            display: flex;
            flex-direction: column;
            align-items: center;
            position: relative;
            width: 100%;
        }

        /* Children Container - The new branch */
        .children-container {
            display: flex;
            flex-wrap: nowrap;
            overflow-x: auto;
            padding: 20px;
            background-color: #F8FBFD;
            border: 1px solid #CCE7F0;
            border-radius: 15px;
            gap: 30px;
            margin-top: 30px;
            position: relative;
            box-shadow: inset 0 2px 5px rgba(0,0,0,0.03);
            padding-bottom: 20px;
            width: fit-content;
            max-width: calc(100vw - 40px);
            align-self: center;
            transition: all 0.3s ease-out;
        }

        /* Line from parent to children container */
        .children-container::before {
            content: '';
            position: absolute;
            top: -30px;
            left: 50%;
            transform: translateX(-50%);
            width: 4px;
            height: 30px;
            background-color: #64748B;
        }

        /* Line from children container to each child (vertical) */
        .children-container > .family-node > .person::before {
            content: '';
            position: absolute;
            top: -20px;
            left: 50%;
            transform: translateX(-50%);
            width: 4px;
            height: 20px;
            background-color: #64748B;
        }

        /* Arrow on top of child card name box */
        .children-container > .family-node > .person::after {
            content: '';
            position: absolute;
            top: -10px;
            left: 50%;
            transform: translateX(-50%) rotate(45deg);
            width: 10px;
            height: 10px;
            background-color: #64748B;
            clip-path: polygon(50% 0%, 0% 100%, 100% 100%);
            z-index: 1;
        }

        /* Style for person cards within the dynamic branch */
        .children-container .person {
            flex-shrink: 0;
            width: 180px;
            min-width: 180px;
            max-width: 180px;
            padding: 15px 20px;
            box-shadow: 0 4px 8px rgba(0,0,0,0.08);
        }
        .children-container .person strong {
            font-size: 1.15em;
        }
        .children-container .person small {
            font-size: 0.85em;
        }
        .children-container .photo-placeholder {
            width: 70px;
            height: 70px;
            margin-bottom: 10px;
        }


        /* Modal specific styles */
        .modal-overlay {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background-color: rgba(0, 0, 0, 0.6);
            display: flex;
            justify-content: center;
            align-items: center;
            z-index: 1000;
        }

        .modal-content {
            background-color: #FFFFFF;
            padding: 35px;
            border-radius: 15px;
            box-shadow: 0 15px 40px rgba(0, 0, 0, 0.25);
            width: 90%;
            max-width: 550px;
            position: relative;
            animation: fadeIn 0.3s ease-out;
        }

        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(-30px); }
            to { opacity: 1; transform: translateY(0); }
        }

        .modal-content h2 {
            font-size: 2em;
            font-weight: 700;
            color: #334155;
            margin-bottom: 30px;
            text-align: center;
        }

        .modal-content label {
            display: block;
            margin-bottom: 10px;
            font-weight: 600;
            color: #475569;
            font-size: 1.05em;
        }

        .modal-content input,
        .modal-content select {
            width: 100%;
            padding: 14px;
            margin-bottom: 25px;
            border: 1px solid #CBD5E0;
            border-radius: 10px;
            font-size: 1.05em;
            color: #334155;
            background-color: #FDFEFF;
            box-sizing: border-box;
            transition: border-color 0.3s ease, box-shadow 0.3s ease;
        }

        .modal-content input:focus,
        .modal-content select:focus {
            border-color: #4299E1;
            box-shadow: 0 0 0 3px rgba(66, 153, 225, 0.2);
            outline: none;
        }

        .modal-content button {
            padding: 14px 30px;
            border-radius: 10px;
            font-size: 1.15em;
            font-weight: 700;
            cursor: pointer;
            transition: background-color 0.3s ease, transform 0.2s ease, box-shadow 0.3s ease;
        }

        .modal-content button[type="submit"] {
            background-color: #10B981;
            color: white;
            border: none;
            box-shadow: 0 4px 8px rgba(16, 185, 129, 0.2);
        }
        .modal-content button[type="submit"]:hover {
            background-color: #059669;
            transform: translateY(-3px);
            box-shadow: 0 6px 12px rgba(16, 185, 129, 0.3);
        }

        .modal-content .close-btn {
            position: absolute;
            top: 20px;
            left: 20px;
            background: none;
            border: none;
            font-size: 2.2em;
            color: #64748B;
            cursor: pointer;
            padding: 8px;
            border-radius: 50%;
            transition: all 0.3s ease;
        }
        .modal-content .close-btn:hover {
            color: #334155;
            background-color: #F0F4F8;
            transform: rotate(90deg);
        }

        /* Custom Alert Modal */
        #custom-alert-modal .modal-content {
            max-width: 450px;
            text-align: center;
        }
        #custom-alert-modal .modal-content h2 {
            margin-bottom: 20px;
            font-size: 1.8em;
        }
        #custom-alert-modal .modal-content p {
            font-size: 1.1em;
            color: #475569;
        }
        #custom-alert-modal .modal-content button {
            background-color: #4299E1;
            color: white;
            border: none;
            margin-top: 30px;
            box-shadow: 0 4px 8px rgba(66, 153, 225, 0.2);
        }
        #custom-alert-modal .modal-content button:hover {
            background-color: #3182CE;
            transform: translateY(-2px);
            box-shadow: 0 6px 12px rgba(66, 153, 225, 0.3);
        }

        /* Custom Confirm Modal */
        #custom-confirm-modal .modal-content {
            max-width: 450px;
            text-align: center;
        }
        #custom-confirm-modal .modal-content h2 {
            font-size: 1.8em;
            margin-bottom: 20px;
        }
        #custom-confirm-modal .modal-content p {
            font-size: 1.1em;
            color: #475569;
        }
        #custom-confirm-modal .modal-content .button-group {
            display: flex;
            justify-content: center;
            gap: 15px;
            margin-top: 30px;
        }
        #custom-confirm-modal .modal-content .button-group button {
            padding: 10px 20px;
            border-radius: 8px;
            font-size: 1em;
            font-weight: 600;
            cursor: pointer;
            transition: all 0.2s ease;
        }
        #custom-confirm-modal .modal-content #custom-confirm-ok-btn {
            background-color: #EF4444; /* Red-500 */
            color: white;
            border: none;
        }
        #custom-confirm-modal .modal-content #custom-confirm-ok-btn:hover {
            background-color: #DC2626; /* Red-600 */
            transform: translateY(-1px);
        }
        #custom-confirm-modal .modal-content #custom-confirm-cancel-btn {
            background-color: #E2E8F0; /* Light gray */
            color: #475569;
            border: 1px solid #CBD5E0;
        }
        #custom-confirm-modal .modal-content #custom-confirm-cancel-btn:hover {
            background-color: #CBD5E0;
            transform: translateY(-1px);
        }

        /* Awaj Info Modal specific styles */
        #awaj-info-modal .modal-content {
            max-width: 400px;
            text-align: center;
        }
        #awaj-info-modal .modal-content h2 {
            font-size: 1.8em;
            margin-bottom: 20px;
        }
        #awaj-info-modal .modal-content p {
            font-size: 1.1em;
            color: #475569;
            line-height: 1.6;
        }

        /* Password Modal specific styles */
        #password-modal .modal-content {
            max-width: 350px;
            text-align: center;
        }
        #password-modal .modal-content h2 {
            font-size: 1.6em;
            margin-bottom: 20px;
        }
        #password-modal .modal-content input[type="password"] {
            margin-bottom: 20px;
            text-align: center;
            letter-spacing: 2px; /* For a password feel */
        }
        #password-modal .modal-content .button-group {
            display: flex;
            justify-content: center;
            gap: 10px;
            margin-top: 10px;
        }
        #password-modal .modal-content .button-group button {
            padding: 10px 20px;
            border-radius: 8px;
            font-size: 1em;
            font-weight: 600;
            cursor: pointer;
            transition: all 0.2s ease;
        }
        #password-modal .modal-content #password-submit-btn {
            background-color: #4299E1;
            color: white;
            border: none;
        }
        #password-modal .modal-content #password-submit-btn:hover {
            background-color: #3182CE;
        }
        #password-modal .modal-content #password-cancel-btn {
            background-color: #E2E8F0;
            color: #475569;
            border: 1px solid #CBD5E0;
        }
        #password-modal .modal-content #password-cancel-btn:hover {
            background-color: #CBD5E0;
        }


        /* --- Mobile-specific adjustments (max-width: 768px) --- */
        @media (max-width: 768px) {
            body {
                padding: 10px;
            }

            /* Header */
            .bg-white.p-6.rounded-xl.shadow-md {
                padding: 12px;
                margin-bottom: 15px;
                gap: 10px;
                flex-direction: column;
            }

            .title-and-about-btn {
                gap: 6px;
            }

            h1 {
                font-size: 1.9em;
            }

            #awaj-info-btn {
                padding: 5px 12px;
                font-size: 0.85em;
                width: fit-content;
            }

            .admin-controls {
                flex-direction: column;
                width: 100%;
                gap: 8px;
            }

            #add-member-btn {
                padding: 10px 20px;
                font-size: 1em;
                width: 100%;
            }

            #user-id-display {
                font-size: 0.8em;
            }

            /* Family Tree Container */
            .family-tree-container {
                padding: 15px;
                min-height: 350px;
                gap: 25px;
            }

            /* Individual Person Card (Main Level) */
            .person {
                padding: 12px 15px;
                width: 98%;
                max-width: 98%;
            }

            .person strong {
                font-size: 1.1em;
            }

            .person small {
                font-size: 0.8em;
            }

            .photo-placeholder {
                width: 60px;
                height: 60px;
                margin-bottom: 8px;
            }

            /* Expand/Collapse Icon */
            .expand-icon {
                width: 20px;
                height: 20px;
                font-size: 1.1em;
                bottom: 3px;
                right: 3px;
            }

            /* Delete Button on Person Card */
            .delete-button {
                width: 20px; /* Smaller for mobile */
                height: 20px;
                font-size: 0.9em; /* Smaller font */
                top: 5px; /* Adjust position */
                left: 5px;
            }

            /* Children Container (Expanded Branch) */
            .children-container {
                margin-top: 20px;
                padding: 15px;
                gap: 12px;
                padding-bottom: 15px;
                max-width: calc(100vw - 30px);
            }

            /* Person Cards within Children Container */
            .children-container > .family-node > .person {
                width: 110px;
                min-width: 110px;
                max-width: 110px;
                padding: 8px 10px;
            }
            .children-container > .family-node > .person strong {
                font-size: 0.85em;
            }
            .children-container > .family-node > .person small {
                font-size: 0.65em;
            }
            .children-container > .family-node > .person .photo-placeholder {
                width: 40px;
                height: 40px;
                margin-bottom: 5px;
            }
            .children-container::before {
                height: 15px;
                top: -15px;
            }
            .children-container > .family-node > .person::before {
                height: 10px;
                top: -10px;
            }
            .children-container > .family-node > .person::after {
                top: -5px;
                width: 6px;
                height: 6px;
            }


            /* Modal specific styles for mobile */
            .modal-content {
                padding: 20px;
                width: 95%;
            }

            .modal-content h2 {
                font-size: 1.5em;
                margin-bottom: 15px;
            }

            .modal-content label {
                font-size: 0.95em;
            }

            .modal-content input,
            .modal-content select {
                padding: 10px;
                margin-bottom: 15px;
                font-size: 0.95em;
            }

            .modal-content button {
                padding: 10px 20px;
                font-size: 1em;
            }

            .modal-content .close-btn {
                font-size: 1.5em;
                top: 8px;
                left: 8px;
            }

            #custom-alert-modal .modal-content,
            #awaj-info-modal .modal-content,
            #custom-confirm-modal .modal-content,
            #password-modal .modal-content { /* Apply to all modals */
                max-width: 90%;
            }
            #awaj-info-modal .modal-content h2,
            #custom-confirm-modal .modal-content h2,
            #password-modal .modal-content h2 {
                font-size: 1.6em;
            }
            #awaj-info-modal .modal-content p,
            #custom-confirm-modal .modal-content p,
            #password-modal .modal-content p {
                font-size: 1em;
            }
            #custom-confirm-modal .modal-content .button-group,
            #password-modal .modal-content .button-group {
                flex-direction: column; /* Stack buttons vertically */
                gap: 10px;
            }
            #custom-confirm-modal .modal-content .button-group button,
            #password-modal .modal-content .button-group button {
                width: 100%; /* Full width for stacked buttons */
            }
        }
    </style>
</head>
<body>
    <div class="w-full max-w-7xl flex flex-col items-center">
        <!-- Header and Admin Controls -->
        <div class="bg-white p-6 rounded-xl shadow-md w-full max-w-4xl mb-8 flex flex-col md:flex-row justify-between items-center gap-4">
            <!-- Title and About Button Container -->
            <div class="title-and-about-btn">
                <h1 class="text-3xl font-bold text-gray-800 mb-0">شجرة عائلة أعواج</h1>
                <button id="awaj-info-btn" class="about-button">حول</button>
            </div>
            
            <!-- Admin Controls (Add Member Button, Admin Toggle, User ID) -->
            <div class="admin-controls">
                <label class="flex items-center space-x-2 cursor-pointer">
                    <input type="checkbox" id="admin-mode-toggle" class="form-checkbox h-5 w-5 text-blue-600 rounded">
                    <span class="text-gray-700 text-sm">وضع المسؤول</span>
                </label>
                <button id="add-member-btn" class="bg-blue-600 hover:bg-blue-700 text-white font-semibold py-3 px-6 rounded-lg shadow-md transition-all duration-300 transform hover:scale-105 hidden">
                    إضافة فرد جديد
                </button>
                <div id="user-id-display" class="text-gray-600 text-sm md:text-base break-all text-center md:text-right">
                    معرف المستخدم: جاري التحميل...
                </div>
            </div>
        </div>

        <!-- Loading Message -->
        <div id="loading-message" class="text-gray-500 text-lg mt-10">
            جاري تحميل شجرة العائلة...
        </div>

        <!-- Family Tree Render Area -->
        <div id="family-tree-render-area" class="family-tree-container">
            <!-- Family members will be rendered here by JavaScript -->
        </div>
    </div>

    <!-- Add New Member Modal -->
    <div id="add-member-modal" class="modal-overlay hidden">
        <div class="modal-content">
            <button id="close-modal-btn" class="close-btn">&times;</button>
            <h2>إضافة فرد جديد</h2>
            <form id="add-member-form">
                <label for="new-member-name">الاسم الكامل:</label>
                <input type="text" id="new-member-name" name="new-member-name" required class="mb-4">

                <label for="new-member-role">الدور في العائلة:</label>
                <select id="new-member-role" name="new-member-role" required class="mb-4">
                    <option value="">اختر دوراً</option>
                    <option value="grandparent">جد/جدة</option>
                    <option value="parent">أب/أم</option>
                    <option value="child">ابن/ابنة</option>
                    <!-- يمكنك إضافة أدوار أخرى مثل "حفيد" -->
                </select>

                <label for="new-member-gender">الجنس:</label>
                <select id="new-member-gender" name="new-member-gender" required class="mb-4">
                    <option value="">اختر الجنس</option>
                    <option value="male">ذكر</option>
                    <option value="female">أنثى</option>
                </select>

                <label for="new-member-birthdate">تاريخ الميلاد (اختياري):</label>
                <input type="date" id="new-member-birthdate" name="new-member-birthdate" class="mb-4">

                <label for="new-member-deathdate">تاريخ الوفاة (اختياري):</label>
                <input type="date" id="new-member-deathdate" name="new-member-deathdate" class="mb-4">

                <label for="new-member-photo">رابط الصورة (URL) (اختياري):</label>
                <input type="url" id="new-member-photo" name="new-member-photo" placeholder="https://example.com/photo.jpg" class="mb-4">

                <label for="new-member-father">الأب (اختياري):</label>
                <select id="new-member-father" name="new-member-father" class="mb-4">
                    <option value="">اختر الأب (اختياري)</option>
                    <!-- Options will be populated by JavaScript -->
                </select>

                <label for="new-member-mother">الأم (اختياري):</label>
                <select id="new-member-mother" name="new-member-mother" class="mb-6">
                    <option value="">اختر الأم (اختياري)</option>
                    <!-- Options will be populated by JavaScript -->
                </select>

                <button type="submit" class="w-full">إضافة الفرد</button>
            </form>
        </div>
    </div>

    <!-- Custom Alert Modal (replaces window.alert) -->
    <div id="custom-alert-modal" class="modal-overlay hidden">
        <div class="modal-content">
            <h2>تنبيه</h2>
            <p id="custom-alert-message" class="text-gray-700"></p>
            <button id="custom-alert-ok-btn">موافق</button>
        </div>
    </div>

    <!-- Custom Confirmation Modal -->
    <div id="custom-confirm-modal" class="modal-overlay hidden">
        <div class="modal-content">
            <h2>تأكيد الحذف</h2>
            <p id="custom-confirm-message" class="text-gray-700"></p>
            <div class="button-group">
                <button id="custom-confirm-ok-btn">تأكيد</button>
                <button id="custom-confirm-cancel-btn">إلغاء</button>
            </div>
        </div>
    </div>

    <!-- Awaj Info Modal -->
    <div id="awaj-info-modal" class="modal-overlay hidden">
        <div class="modal-content">
            <button id="close-awaj-info-modal-btn" class="close-btn">&times;</button>
            <h2>معلومات عن شجرة عائلة أعواج</h2>
            <p class="text-gray-700">تم تصميم هذه الشجرة من قبل محمد الديب تحت إشراف محمود.</p>
        </div>
    </div>

    <!-- Password Modal -->
    <div id="password-modal" class="modal-overlay hidden">
        <div class="modal-content">
            <h2>أدخل كلمة المرور</h2>
            <input type="password" id="admin-password-input" placeholder="كلمة المرور" class="mb-4">
            <div class="button-group">
                <button id="password-submit-btn">دخول</button>
                <button id="password-cancel-btn">إلغاء</button>
            </div>
        </div>
    </div>

</body>
</html>
