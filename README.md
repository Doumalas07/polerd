<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>PÔLE R&D</title>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/bootstrap/5.1.3/css/bootstrap.min.css">
    <script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
    <script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f3f3f3;
            color: #333;
        }
        h1, h2 {
            color: #007bff; /* Bleu */
        }
        .progress-container {
            width: 100%;
            background-color: #e0e0e0;
            border-radius: 8px;
            overflow: hidden;
            margin-top: 20px;
        }
        .progress-bar {
            height: 20px;
            background-color: #ff5722; /* Orange */
            width: 0%;
            transition: width 0.3s ease;
        }
        #non-conformity-container {
            margin-top: 20px;
        }
        /* Tableau stylisé */
        #nonConformityTable {
            border-collapse: collapse;
            width: 100%;
            background-color: #ffffff;
        }
        #nonConformityTable thead {
            background-color: #007bff; /* Bleu */
            color: #ffffff;
        }
        #nonConformityTable th, #nonConformityTable td {
            padding: 12px;
            text-align: center;
            border: 1px solid #e0e0e0;
        }
        #nonConformityTable tbody tr:nth-child(even) {
            background-color: #f9f9f9;
        }
        #nonConformityTable tbody tr:hover {
            background-color: #ffebd9; /* Orange clair */
        }
        #nonConformityTable tbody tr td {
            color: #333;
        }
        #nonConformityTable th {
            font-weight: bold;
        }
        @media (max-width: 576px) {
            h1, h2 {
                font-size: 1.5rem;
            }
        }
    </style>
</head>
<body>
    <div class="container mt-5">
        <h1 class="text-center">PÔLE R&D - DIRECTION DES OPERATIONS</h1>
        <h2 class="text-center">Inconformité Déclarations Carburant & Gaz</h2>
        
        <!-- Barre de progression -->
        <div class="progress-container">
            <div class="progress-bar" id="progressBar"></div>
        </div>

        <div class="my-4 text-center">
            <input type="file" id="inputFile" class="form-control" accept=".xlsx, .xls">
        </div>

        <div id="non-conformity-container" class="mt-5">
            <h3>Non-Conformités par date</h3>
            <table id="nonConformityTable" class="table table-bordered">
                <thead>
                    <tr>
                        <th>Immatriculation Tracteur</th>
                        <th>Numéro BL</th>
                        <th>Date de Chargement</th>
                        <th>Date de Livraison</th>
                        <th>Date Non-Conforme</th>
                    </tr>
                </thead>
                <tbody>
                </tbody>
            </table>
        </div>
    </div>

    <script>
        document.getElementById('inputFile').addEventListener('change', handleFile, false);

        function updateProgressBar(percentage) {
            const progressBar = document.getElementById('progressBar');
            progressBar.style.width = percentage + '%';
        }

        function handleFile(e) {
            updateProgressBar(0); // Reset progress bar at start
            const file = e.target.files[0];
            const reader = new FileReader();
            reader.onloadstart = () => updateProgressBar(25);
            reader.onload = function(e) {
                updateProgressBar(50);
                const data = new Uint8Array(e.target.result);
                const workbook = XLSX.read(data, { type: 'array' });
                const firstSheet = workbook.Sheets[workbook.SheetNames[0]];
                const jsonData = XLSX.utils.sheet_to_json(firstSheet, { header: 1 });
                analyzeData(jsonData);
                updateProgressBar(100); // Completion
            };
            reader.readAsArrayBuffer(file);
        }

        function analyzeData(data) {
            const headers = data[0];
            const rows = data.slice(1);
            const conformityData = [];
            const tracteurs = {};
            const immatriculationIndex = headers.indexOf('tracteur');
            const dateChargementIndex = headers.indexOf('dateChargement');
            const dateLivraisonIndex = headers.indexOf('dateLivraison');
            const numblIndex = headers.indexOf('numbl');
            const depotlibIndex = headers.indexOf('depotlib');

            rows.forEach(row => {
                const tracteur = row[immatriculationIndex];
                const dateChargement = new Date(row[dateChargementIndex]);
                const dateLivraison = new Date(row[dateLivraisonIndex]);
                const numbl = row[numblIndex];
                const depotlib = row[depotlibIndex];

                if (!tracteurs[tracteur]) {
                    tracteurs[tracteur] = [];
                }
                tracteurs[tracteur].push({ dateChargement, dateLivraison, numbl, depotlib });
            });

            for (const tracteur in tracteurs) {
                const deliveries = tracteurs[tracteur];
                deliveries.sort((a, b) => a.dateChargement - b.dateChargement);

                for (let i = 0; i < deliveries.length - 1; i++) {
                    const currentDelivery = deliveries[i];
                    const nextDelivery = deliveries[i + 1];
                    if (
                        nextDelivery.dateChargement > currentDelivery.dateChargement &&
                        nextDelivery.dateChargement < currentDelivery.dateLivraison
                    ) {
                        conformityData.push({
                            tracteur,
                            numbl: nextDelivery.numbl,
                            dateChargement: nextDelivery.dateChargement.toLocaleDateString(),
                            dateLivraison: currentDelivery.dateLivraison.toLocaleDateString(),
                            dateNonConforme: nextDelivery.dateChargement.toLocaleDateString()
                        });
                    }
                }
            }

            populateNonConformityTable(conformityData);
        }

        function populateNonConformityTable(data) {
            const tableBody = document.querySelector('#nonConformityTable tbody');
            tableBody.innerHTML = '';
            data.forEach(item => {
                const row = `<tr>
                    <td>${item.tracteur}</td>
                    <td>${item.numbl}</td>
                    <td>${item.dateChargement}</td>
                    <td>${item.dateLivraison}</td>
                    <td>${item.dateNonConforme}</td>
                </tr>`;
                tableBody.innerHTML += row;
            });
        }
    </script>
</body>
</html>
