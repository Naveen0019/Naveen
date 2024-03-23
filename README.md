%code for lte waveform
enb.NDLRB = 6;                            % No of Downlink Resource Blocks(DL-RB)
enb.CyclicPrefix = 'Normal';          % CP length
enb.PHICHDuration = 'Normal';    % Normal PHICH duration
enb.DuplexMode = 'FDD';             % FDD duplex mode
enb.CFI = 3;                    % 4 PDCCH symbols
enb.Ng = 'Sixth';              % HICH groups
enb.CellRefP = 4;            % 4-antenna ports
enb.NCellID = 10;           % Cell id
enb.NSubframe = 0;        % Subframe number 0

subframe = lteDLResourceGrid(enb);
pdsch.NLayers = 4;                              % No of layers
pdsch.TxScheme = 'TxDiversity';        % Transmission scheme
pdsch.Modulation = 'QPSK';                % Modulation scheme
pdsch.RNTI = 1;                                   % 16-bit UE-specific mask
pdsch.RV = 0;                                       % Redundancy Version

pdsch.PRBSet = (0:enb.NDLRB-1).';   % Subframe resource allocation
[pdschIndices,pdschInfo] = ...
    ltePDSCHIndices(enb, pdsch, pdsch.PRBSet, {'1based'});
codedTrBlkSize = pdschInfo.G;   % Available PDSCH bits

transportBlkSize = 152;                % Transport block size
dlschTransportBlk = randi([0 1], transportBlkSize, 1);

% Perform Channel Coding
codedTrBlock = lteDLSCH(enb, pdsch, codedTrBlkSize, ...
               dlschTransportBlk);
pdschSymbols = ltePDSCH(enb, pdsch, codedTrBlock);
subframe(pdschIndices) = pdschSymbols;
dci.DCIFormat = 'Format1A';  % DCI message format
dci.Allocation.RIV = 26;         % Resource indication value

[dciMessage, dciMessageBits] = lteDCI(enb, dci); % DCI message
pdcch.NDLRB = enb.NDLRB;  % Number of DL-RB in total BW
pdcch.RNTI = pdsch.RNTI;  % 16-bit value number
pdcch.PDCCHFormat = 0;    % 1-CCE of aggregation level 1



10


% Performing DCI message bits coding to form coded DCI bits
codedDciBits = lteDCIEncode(pdcch, dciMessageBits);
pdcchInfo = ltePDCCHInfo(enb);    % Get the total resources for PDCCH
pdcchBits = -1*ones(pdcchInfo.MTot, 1); % Initialized with -1

% Performing search space for UE-specific control channel candidates
candidates = ltePDCCHSpace(enb, pdcch, {'bits','1based'});

% Mapping PDCCH payload on available UE-specific candidate. In this example
% the first available candidate is used to map the coded DCI bits.
pdcchBits( candidates(1, 1) : candidates(1, 2) ) = codedDciBits;
pdcchSymbols = ltePDCCH(enb, pdcchBits);
pdcchIndices = ltePDCCHIndices(enb, {'1based'});

% The complex PDCCH symbols are easily mapped to each of the resource grids
% for each antenna port
subframe(pdcchIndices) = pdcchSymbols;
cfiBits = lteCFI(enb);
pcfichSymbols = ltePCFICH(enb, cfiBits);
pcfichIndices = ltePCFICHIndices(enb);

% Map PCFICH symbols to resource grid
subframe(pcfichIndices) = pcfichSymbols;
surf(abs([subframe(:,:,1);subframe(end,:,1)]));
view(2);
h = rotate3d; setAllowAxesRotate(h,gca,false);
axis tight;
xlabel('OFDM Symbol');
ylabel('Subcarrier');
title('Resource grid');
