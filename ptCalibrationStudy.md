* per lanciare i boost ho inserito in .bashrc 
* export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/afs/cern.ch/cms/slc5_amd64_gcc434/external/boost/1.47.0/lib

# Da ZFitter: (cd ECALELF_run2) 
 
* Prima di incominciare proprio, chiarisci a te stesso quale variabile di energia vuoi utilizzare 
# invMass_var=invMass_SC_regrCorrSemiParV5_ele 

# L'energia giusta va specificata A MANO in src/ElectronCategory_class.cc 
(FALLO, altrimenti categorizzi con l'energia sbagliata) 

# Aggiungere il branch ZPt (mi serve perche' ci categorizzo sopra) 
./bin/ZFitter.exe -f data/validation/22Jan2012-runDepMCAll_checkMee.dat --regionsFile=data/regions/scaleStep0.dat --addBranch=ZPt_energySCEle_regrCorrSemiParV5_ele --corrEleType=HggRunEtaR9Et --smearEleType=stochastic --saveRootMacro &> debug.txt
./script/hadder.sh
mv tmp/ZPt_energySCEle_* friends/other/ 
* Aggiungi i branch di ZPt nel config file 

# Categorizza 
./script/GenRootChain.sh doesn't accept smearing??

./bin/ZFitter.exe -f data/validation/22Jan2012-runDepMCAll_checkMee.dat --regionsFile=data/regions/scaleStep0.dat --addBranch=smearerCat --corrEleType=HggRunEtaR9Et --smearEleType=stochastic invMass_var=invMass_SC_regrCorrSemiParV5_ele --saveRootMacro
./script/hadder.sh

* Controllare che la categorizzazione sia fatta bene: 
Devi associare i friend trees con tmp/load.C. A questo punto puoi fare plot in questo modo, chiedendo branch da tree diversi 
root -l tmp/d_chain.root tmp/load_singleFile.C 
data->Draw("ZPta_energySCEle_regrCorrSemiParV5_ele","smearerCat[0]>0") 
data->Draw("invMass_SC_regrCorrSemiParV5_ele","smearerCat[0]>0") 

mv tmp/smearerCat_scaleStep0* friends/smearerCat/ 
* Aggiungi i branch di categoria nel config file 

# QUI SEI PRONTO A RIEMPIRE GLI ISTOGRAMMI CON LA TARGET VARIABLE E MINIMIZZARE LA LIKELIHOOD 
* In minimizer.sh c'e' tutta la storia
* Attento che in RooSmearer::SetSmearedHisto ho eliminato lo smoothing artificiale smearedHisto->Smooth() 

# Minimizzare la likelihood (lo faccio 50 volte)
./script/run50Minimization.sh ptRatio (opp ptRatio_ranom opp ptSum)
./script/Likelihodd_fitter.sh ptRatio

# Riempire solo gli istogrammi 
./bin/ZFitter.exe -f data/validation/22Jan2012-runDepMCAll_checkMee.dat --regionsFile=data/regions/scaleStep0.dat --invMass_var=invMass_SC_regrCorrSemiParV5_ele --autoBin --smearerFit --plotOnly --targetVariable=ptRatio --targetVariable_min=0.5 --targetVariable_max=2 --targetVariable_binWidth=0.05 --configuration=leading --corrEleType=HggRunEtaR9Et --smearEleType=stochastic &> tmp/debug.txt 

root -l test/dato/fitres/histos_scaleStep0_Et_25_trigger_noPF.root 

# FARE UN RAPIDO PROFILO DELLA LIKELIHOOD (Prende i profili attorno ai valori iniziali dei parametri, non fa LA vera minimizzazione) 
./bin/ZFitter.exe -f data/validation/22Jan2012-runDepMCAll_checkMee.dat --regionsFile=data/regions/scaleStep0.dat --invMass_var=invMass_SC_regrCorrSemiParV5_ele --autoBin --smearerFit --plotOnly --profileOnly --targetVariable=ptRatio --targetVariable_min=0.5 --targetVariable_max=2 --targetVariable_binWidth=0.05 --corrEleType=HggRunEtaR9Et --smearEleType=stochastic &> tmp/debug.txt 

root -l test/dato/fitres/outProfile_scaleStep0_Et_25_trigger_noPF.root 

# MINIMIZZA la likelihood (ci mette 30 minuti circa)! 
./bin/ZFitter.exe -f data/validation/22Jan2012-runDepMCAll_checkMee.dat --regionsFile=data/regions/scaleStep0.dat --invMass_var=invMass_SC_regrCorrSemiParV5_ele --autoBin --smearerFit --targetVariable=ptRatio --targetVariable_min=0.5 --targetVariable_max=2 --targetVariable_binWidth=0.05 --configuration=leading --corrEleType=HggRunEtaR9Et --smearEleType=stochastic &> tmp/debug_ptRatio_leading.txt 

rm -r tmp/tmpFile-*.root 

sshfs gfasanel@lxplus6.cern.ch:/afs/cern.ch/user/g/gfasanel/new_version_ECALELF/CMSSW_7_5_0_pre4/src/Calibration/ZFitter/test/dato/img/ Scrivania/remote_dir/ 

# PLOT DATA MC FINALI per vedere se la minimizzazione della likelihood ha avuto successo e ficcare questi plot nelle slide 
cp test/dato/fitres/params-scaleStep0-Et_25-trigger-noPF.txt output_pt12.txt 
rm test/dato/fitres/histos_ptRatio_leading_scaleStep0_Et_25_trigger_noPF.root 
./bin/ZFitter.exe -f data/validation/22Jan2012-runDepMCAll_checkMee.dat --regionsFile=data/regions/scaleStep0.dat --invMass_var=invMass_SC --autoBin --smearerFit –initFile=output_pt12.txt --targetVariable=ptRatio --configuration=leading --targetVariable_min=0.5 --targetVariable_max=2 --targetVariable_binWidth=0.05 --plotOnly --nSmearToy=1 --outDirImgData=test/dato/img --outDirFitResData=test/dato/fitres 

root -l -b 
.L macro/plot_data_mc.C+ 
PlotMeanHist("test/dato/fitres/histos_ptRatio_leading_scaleStep0_Et_25_trigger_noPF.root") 
.q 

# Plottare 1 solo profile della likelihood
root -l -b
.L tmp/fitOneProfile.C
fitOneProfile("profile.root","outDir")


## DRITTE
DRITTE PER AIUTARE LA MINIMIZZAZIONE SE DOVESSE FALLIRE AL PRIMO COLPO 
cp test/dato/fitres/params-scaleStep0-Et_25-noPF.txt init_pt12.txt 
e modifichi init_pt12.dat per avvicinare i parametri iniziali a quelli veri (allarghi i range se necessario) 

A questo punto rigiri con quell'initFile e cerchi di imboccarlo 
./bin/ZFitter.exe -f data/validation/monitoring_2012_53X.dat --regionsFile=data/regions/scaleStep0.dat --autoBin --smearerFit --plotOnly --profileOnly --noPU --commonCut=Et_25-noPF --targetVariable=ptRatio --targetVariable_min=0 --targetVariable_max=5 –initFile=init_pt12.txt &>debug.txt 


# emacVARIABILE SCALARE
PtSUM 
./bin/ZFitter.exe -f data/validation/22Jan2012-runDepMCAll_checkMee.dat --regionsFile=data/regions/scaleStep0.dat --invMass_var=invMass_SC --autoBin --smearerFit  --targetVariable=ptSum --targetVariable_min=0 --targetVariable_max=200 --targetVariable_binWidth=1 --corrEleType=HggRunEtaR9Et –smearEleType=stochastic &>debug.txt 
