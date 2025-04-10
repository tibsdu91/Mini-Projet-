// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.24;

import "./Ownable.sol";
import "./SafeMath.sol";

contract Election is Ownable {

    using SafeMath for uint256;

    // Enum pour les types de vote
    enum VoteOption { POUR, CONTRE, NEUTRE }

    // Modèle d'une résolution
    struct Resolution {
        uint256 id;
        string title;
        uint voteCountPour;
        uint voteCountContre;
        uint voteCountNeutre;
        bool isActive;
    }

    // Liste blanche des participants autorisés à voter
    mapping(address => bool) public whiteList;
    
    // Comptabiliser les votants pour chaque résolution
    mapping(uint => mapping(address => bool)) public hasVoted;

    // Stocker les résolutions
    mapping(uint => Resolution) public resolutions;

    // Compter les résolutions
    uint public resolutionsCount;

    // Rôles des électeurs
    address public president; // Président de séance
    address public scrutateur; // Scrutateur
    address public secretaire; // Secrétaire (Owner du contrat)

    // Événement de vote
    event VotedEvent(uint indexed _resolutionId, VoteOption _voteOption);
    event WhiteListUpdated(address _voter, bool _status);
    event ResolutionAdded(uint _resolutionId, string _title);
    event RolesUpdated(address _president, address _scrutateur);

    // Modifier pour vérifier si l'adresse est dans la liste blanche
    modifier onlyWhiteListed() {
        require(whiteList[msg.sender], "Vous n'etes pas autorise a voter.");
        _;
    }

    // Modifier les fonctions pour vérifier les rôles
    modifier onlyPresident() {
        require(msg.sender == president, "Vous devez etre le president de seance pour cette action.");
        _;
    }

    modifier onlyScrutateur() {
        require(msg.sender == scrutateur, "Vous devez etre le scrutateur pour cette action.");
        _;
    }

    // Déployer avec les rôles (le secrétaire étant le owner)
    constructor(address _president, address _scrutateur) {
        president = _president;
        scrutateur = _scrutateur;
        secretaire = owner; // Le créateur du contrat est le secrétaire
        
        // Ajouter automatiquement les rôles à la liste blanche
        whiteList[president] = true;
        whiteList[scrutateur] = true;
        whiteList[secretaire] = true;
    }

    // Ajouter une résolution (uniquement par le président)
    function addResolution(string memory _title) public onlyPresident {
        resolutionsCount++;
        resolutions[resolutionsCount] = Resolution(resolutionsCount, _title, 0, 0, 0, true);
        emit ResolutionAdded(resolutionsCount, _title);
    }

    // Gestion de la liste blanche des participants (ajout/suppression)
    function updateWhiteList(address _voter, bool _status) public onlyOwner {
        whiteList[_voter] = _status;
        emit WhiteListUpdated(_voter, _status);
    }

    // Voter pour une résolution avec un type de vote spécifique
    function vote(uint _resolutionId, VoteOption _voteOption) public onlyWhiteListed {
        // Vérifier que l'ID de la résolution est valide
        require(_resolutionId > 0 && _resolutionId <= resolutionsCount, "ID de resolution invalide");
        require(resolutions[_resolutionId].isActive, "Cette resolution n'est plus active");
        
        // Vérifier que l'utilisateur n'a pas déjà voté pour cette résolution
        require(!hasVoted[_resolutionId][msg.sender], "Vous avez deja vote pour cette resolution.");

        // Enregistrer que l'électeur a voté pour cette résolution
        hasVoted[_resolutionId][msg.sender] = true;

        // Mettre à jour le compteur de votes en fonction du type de vote
        if (_voteOption == VoteOption.POUR) {
            resolutions[_resolutionId].voteCountPour++;
        } else if (_voteOption == VoteOption.CONTRE) {
            resolutions[_resolutionId].voteCountContre++;
        } else if (_voteOption == VoteOption.NEUTRE) {
            resolutions[_resolutionId].voteCountNeutre++;
        }

        // Déclencher l'événement de vote
        emit VotedEvent(_resolutionId, _voteOption);
    }

    // Obtenir les résultats des votes d'une résolution
    function getVoteCounts(uint _resolutionId) public view returns (uint pour, uint contre, uint neutre) {
        require(_resolutionId > 0 && _resolutionId <= resolutionsCount, "ID de resolution invalide");
        Resolution storage resolution = resolutions[_resolutionId];
        return (resolution.voteCountPour, resolution.voteCountContre, resolution.voteCountNeutre);
    }
    
    // Clôturer une résolution (uniquement par le scrutateur)
    function closeResolution(uint _resolutionId) public onlyScrutateur {
        require(_resolutionId > 0 && _resolutionId <= resolutionsCount, "ID de resolution invalide");
        resolutions[_resolutionId].isActive = false;
    }

    // Permet de mettre à jour les rôles (seulement par le propriétaire du contrat, le secrétaire)
    function updateRoles(address _president, address _scrutateur) public onlyOwner {
        president = _president;
        scrutateur = _scrutateur;
        emit RolesUpdated(_president, _scrutateur);
    }
    
    // Transférer la propriété du contrat (changement de secrétaire)
    function transferSecretary(address _newSecretary) public onlyOwner {
        transferOwnership(_newSecretary);
        secretaire = _newSecretary;
        whiteList[_newSecretary] = true; // Ajouter le nouveau secrétaire à la liste blanche
    }
}