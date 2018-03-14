#!/bin/bash

#   Copyright (C) 2004 University of Oxford
#
#   SHCOPYRIGHT
export LC_ALL=C

export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:${FSLDIR}/lib

Usage() {
    echo ""
    echo "Usage: bedpostx <subject_directory> [options]"
    echo ""
    echo "expects to find bvals and bvecs in subject directory"
    echo "expects to find data and nodif_brain_mask in subject directory"
    echo "expects to find grad_dev in subject directory, if -g is set"
    echo ""
    echo "<options>:"
    #echo "-QSYS (Queue System, 0 use fsl_sub: FMRIB, 1 TORQUE (default): WashU)"
    echo "-Q (name of the GPU(s) queue, default cuda.q (defined in environment variable: FSLGECUDAQ)"
    #echo "-Q (name of the GPU(s) queue, default cuda.q for QSYS=0 and no queue for QSYS=1)"
    echo "-NJOBS (number of jobs to queue, the data is divided in NJOBS parts, usefull for a GPU cluster, default 4)"
    echo "-n (number of fibres per voxel, default 3)"
    echo "-w (ARD weight, more weight means less secondary fibres per voxel, default 1)"
    echo "-b (burnin period, default 1000)"
    echo "-j (number of jumps, default 1250)"
    echo "-s (sample every, default 25)"
    echo "-model (Deconvolution model. 1: with sticks, 2: with sticks with a range of diffusivities (default), 3: with zeppelins)"
    echo "-g (consider gradient nonlinearities, default off)"
    echo ""
    echo ""
    echo "ALTERNATIVELY: you can pass on xfibres options onto directly bedpostx"
    echo " For example:  bedpostx <subject directory> --noard --cnonlinear"
    echo " Type 'xfibres --help' for a list of available options "
    echo " Default options will be bedpostx default (see above), and not xfibres default."
    echo ""
    echo "Note: Use EITHER old OR new syntax."
    exit 1
}

monitor(){
    cat <<EOM > ${subjdir}.bedpostX/monitor
#!/bin/sh
nparts=0
if [ $njobs -eq 1 ]; then
#1 part (GPU) and several subparts
#voxels processed in each subpart are 12800 or more if the last one is less than 6400 (1 part less)
	nparts=\$(($nvox/12800))
	if [ \$nparts%12800 != 0 ];then
		nparts=\$((\$nparts + 1))
	fi
	last_part=\$(($nvox-(((\$nparts-1))*12800)))
	if [ \$last_part -lt 6400 ];then
		nparts=\$((\$nparts - 1))
	fi
else
	nparts=$njobs
fi

echo
echo "----- Bedpostx Monitor -----"
finished=0
lastprinted=0
havedad=2
while [ \$finished -eq 0 ] ; do
    nfin=0
    part=0
    errorFiles=\`ls ${subjdir}.bedpostX/logs/*.e* 2> /dev/null \`
    for errorFile in \$errorFiles
    do
        if [ -s \$errorFile ]; then
            echo An error ocurred. Please check file \$errorFile
            kill -9 $$
            exit 1
        fi
    done
    while [ \$part -le \$nparts ];do
        if [ -e ${subjdir}.bedpostX/logs/monitor/\$part ]; then
            nfin=\$((\$nfin + 1))
        fi
        part=\$((\$part + 1))
    done
    newmessages=\$((\$nfin - \$lastprinted))
    while [ "\$newmessages" -gt 0 ];do
        lastprinted=\$((\$lastprinted + 1))
        echo \$lastprinted parts processed out of \$nparts
        newmessages=\$((\$newmessages - 1))
    done
    if [ -f ${subjdir}.bedpostX/xfms/eye.mat ] ; then
        finished=1
        echo "All parts processed"
	exit
    fi
    if [ ! \$havedad -gt 0 ]; then
       exit 0
    fi
    if [ "x$SGE_ROOT" = "x" ]; then
        havedad=\`ps -e -o pid 2>&1| grep "$$\\b" | wc -l\`
    fi
    sleep 50;
done
EOM
    chmod +x ${subjdir}.bedpostX/monitor
}

make_absolute(){
    dir=$1;
    if [ -d ${dir} ]; then
	OLDWD=`pwd`
	cd ${dir}
	dir_all=`pwd`
	cd $OLDWD
    else
	dir_all=${dir}
    fi
    echo ${dir_all}
}

[ "$1" = "" ] && Usage

subjdir=`make_absolute $1`
subjdir=`echo $subjdir | sed 's/\/$/$/g'`

echo "---------------------------------------------"
echo "------------ BedpostX GPU Version -----------"
echo "---------------------------------------------"
echo subjectdir is $subjdir

#parse option arguments
qsys=0
njobs=4
nfibres=3
fudge=1
burnin=1000
njumps=1250
sampleevery=25
model=2
gflag=0
other=""
queue=""
slurm=0

if [ $qsys -eq 0 ] && [ "x$SGE_ROOT" != "x" ]; then
	queue="-q $FSLGECUDAQ"
fi

shift
while [ ! -z "$1" ]
do
  case "$1" in
      -QSYS) qsys=$2;shift;;
      -Q) queue="-q $2";shift;;
      -NJOBS) njobs=$2;shift;;
      -n) nfibres=$2;shift;;
      -w) fudge=$2;shift;;
      -b) burnin=$2;shift;;
      -j) njumps=$2;shift;;
      -s) sampleevery=$2;shift;;
      -model) model=$2;shift;;
      -g) gflag=1;;
      *) other=$other" "$1;;
  esac
  shift
done
opts="--nf=$nfibres --fudge=$fudge --bi=$burnin --nj=$njumps --se=$sampleevery --model=$model"
defopts="--cnonlinear"
opts="$opts $defopts $other"

#Check for Slurm
echo "Locating CUDA queue."
if [ -n "$FSLGECUDAQ" ]; then
  sinfo --hide --partition=$FSLGECUDAQ 2>&1 >/dev/null
	if [ $? -eq 0 ]; then
      slurm=1
	fi
fi

#check that all required files exist

if [ ! -d $subjdir ]; then
	echo "subject directory $1 not found"
	exit 1
fi

if [ ! -e ${subjdir}/bvecs ]; then
  if [ "$(ls ${subjdir}/*bvec* | wc -l)" -ge "1" ]; then ## Rework for BIDs
    mv ${subjdir}/*bvec* ${subjdir}/bvecs
  else
    echo "${subjdir}/bvecs not found"
    exit 1
  fi
fi

if [ ! -e ${subjdir}/bvals ]; then
  if [ "$(ls ${subjdir}/*bval* | wc -l)" -ge "1" ]; then ## Rework for BIDs
    mv ${subjdir}/*bval* ${subjdir}/bvals
  else
    echo "${subjdir}/bvals not found"
    exit 1
  fi
fi

if [ `${FSLDIR}/bin/imtest ${subjdir}/data` -eq 0 ]; then
	echo "${subjdir}/data not found"
	exit 1
fi

if [ ${gflag} -eq 1 ]; then
    if [ `${FSLDIR}/bin/imtest ${subjdir}/grad_dev` -eq 0 ]; then
	echo "${subjdir}/grad_dev not found"
	exit 1
    fi
fi

if [ `${FSLDIR}/bin/imtest ${subjdir}/nodif_brain_mask` -eq 0 ]; then
	echo "${subjdir}/nodif_brain_mask not found"
	exit 1
fi

if [ -e ${subjdir}.bedpostX/xfms/eye.mat ]; then
	echo "${subjdir} has already been processed: ${subjdir}.bedpostX."
	echo "Delete or rename ${subjdir}.bedpostX before repeating the process."
	exit 1
fi

echo Making bedpostx directory structure

mkdir -p ${subjdir}.bedpostX/
mkdir -p ${subjdir}.bedpostX/diff_parts
mkdir -p ${subjdir}.bedpostX/logs
mkdir -p ${subjdir}.bedpostX/logs/logs_gpu
mkdir -p ${subjdir}.bedpostX/logs/monitor
rm -f ${subjdir}.bedpostX/logs/monitor/*
mkdir -p ${subjdir}.bedpostX/xfms

#mailto=`whoami`@fmrib.ox.ac.uk

echo Copying files to bedpost directory

cp ${subjdir}/bvecs ${subjdir}/bvals ${subjdir}.bedpostX
${FSLDIR}/bin/imcp ${subjdir}/nodif_brain_mask ${subjdir}.bedpostX
if [ `${FSLDIR}/bin/imtest ${subjdir}/nodif` = 1 ] ; then
    ${FSLDIR}/bin/fslmaths ${subjdir}/nodif -mas ${subjdir}/nodif_brain_mask ${subjdir}.bedpostX/nodif_brain
fi


# Split the dataset in parts
echo Pre-processing stage

if [ ${gflag} -eq 1 ]; then
	pre_command="$FSLDIR/bin/split_parts_gpu ${subjdir}/data ${subjdir}/nodif_brain_mask ${subjdir}.bedpostX/bvals ${subjdir}.bedpostX/bvecs ${subjdir}/grad_dev 1 $njobs ${subjdir}.bedpostX"
else
	pre_command="$FSLDIR/bin/split_parts_gpu ${subjdir}/data ${subjdir}/nodif_brain_mask ${subjdir}.bedpostX/bvals ${subjdir}.bedpostX/bvecs NULL 0 $njobs ${subjdir}.bedpostX"
fi
if [ $qsys -eq 0 ] && [ $slurm -eq 0]; then
	#SGE
	splitID=`${FSLDIR}/bin/fsl_sub -T 40 -l ${subjdir}.bedpostX/logs -N bedpostx_preproc_gpu $pre_command`
elif [ $qsys -eq 0 ] && [ $slurm -eq 1 ]; then
  #SLURM
  splitID=`${FSLDIR}/bin/fsl_sub -F -T 40 -l ${subjdir}.bedpostX/logs -N bedpostx_preproc_gpu $pre_command | grep -oP "Submitted batch job\s+\K\w+"`
else
	#TORQUE
	echo $pre_command > ${subjdir}.bedpostX/temp
	torque_command="qsub -V $queue -l nodes=1:ppn=1:gpus=1,walltime=00:40:00 -N bedpostx_preproc_gpu -o ${subjdir}.bedpostX/logs -e ${subjdir}.bedpostX/logs"
	splitID=`exec $torque_command ${subjdir}.bedpostX/temp | awk '{print $1}' | awk -F. '{print $1}'`
        rm ${subjdir}.bedpostX/temp
	sleep 10
fi


nvox=`${FSLDIR}/bin/fslstats $subjdir.bedpostX/nodif_brain_mask -V  | cut -d ' ' -f1 `

echo Queuing parallel processing stage

[ -f ${subjdir}.bedpostX/commands.txt ] && rm ${subjdir}.bedpostX/commands.txt

monitor
if [ "x$SGE_ROOT" = "x" ]; then
    ${subjdir}.bedpostX/monitor&
fi

part=0
while [ $part -lt $njobs ]
do
    	partzp=`$FSLDIR/bin/zeropad $part 4`

	if [ ${gflag} -eq 1 ]; then
	    gopts="$opts --gradnonlin=${subjdir}.bedpostX/grad_dev_$part"
	else
	    gopts=$opts
	fi

	echo "${FSLDIR}/bin/xfibres_gpu --data=${subjdir}.bedpostX/data_$part --mask=$subjdir.bedpostX/nodif_brain_mask -b ${subjdir}.bedpostX/bvals -r ${subjdir}.bedpostX/bvecs --forcedir --logdir=$subjdir.bedpostX/diff_parts/data_part_$partzp $gopts ${subjdir} $part $njobs $nvox" >> ${subjdir}.bedpostX/commands.txt

    	part=$(($part + 1))
done

if [ $qsys -eq 0 ] && [ $slurm -eq 0]; then
	#SGE
	bedpostid=`${FSLDIR}/bin/fsl_sub $queue -l ${subjdir}.bedpostX/logs -N bedpostx_gpu -j $splitID -t ${subjdir}.bedpostX/commands.txt`
elif [ $qsys -eq 0 ] && [ $slurm -eq 1 ]; then
  #SLURM
  bedpostid=`${FSLDIR}/bin/fsl_sub $queue -l ${subjdir}.bedpostX/logs -N bedpostx_gpu -j $splitID -t ${subjdir}.bedpostX/commands.txt | grep -oP "Submitted batch job\s+\K\w+"`
else
	#TORQUE
	taskfile=${subjdir}.bedpostX/commands.txt
	echo "command=\`cat "$taskfile" | head -\$PBS_ARRAYID | tail -1\` ; exec \$command" > ${subjdir}.bedpostX/temp
	tasks=`wc -l $taskfile | awk '{print $1}'`
	sge_tasks="-t 1-$tasks"
	#PBS -t x-y: x and y are the array bounds
	torque_command="qsub -V $queue -l nodes=1:ppn=1:gpus=1,walltime=3:00:00,pmem=16gb -N bedpostx_gpu -o ${subjdir}.bedpostX/logs -e ${subjdir}.bedpostX/logs -W depend=afterok:$splitID $sge_tasks"
	bedpostid=`exec $torque_command ${subjdir}.bedpostX/temp | awk '{print $1}' | awk -F. '{print $1}'`
        rm ${subjdir}.bedpostX/temp
	sleep 10
fi


echo Queuing post processing stage
post_command="${FSLDIR}/bin/bedpostx_postproc_gpu.sh --data=${subjdir}/data --mask=$subjdir.bedpostX/nodif_brain_mask -b ${subjdir}.bedpostX/bvals -r ${subjdir}.bedpostX/bvecs  --forcedir --logdir=$subjdir.bedpostX/diff_parts $gopts $nvox $njobs ${subjdir} ${FSLDIR}"
if [ $qsys -eq 0 ] && [ $slurm -eq 0]; then
	#SGE
	mergeid=`${FSLDIR}/bin/fsl_sub -T 120 -j $bedpostid -N bedpostx_postproc_gpu -l ${subjdir}.bedpostX/logs $post_command`
elif [ $qsys -eq 0 ] && [ $slurm -eq 1 ]; then
  #SLURM
  mergeid=`${FSLDIR}/bin/fsl_sub -T 120 -j $bedpostid -N bedpostx_postproc_gpu -l ${subjdir}.bedpostX/logs $post_command | grep -oP "Submitted batch job\s+\K\w+"`
else
	#TORQUE
	echo $post_command > ${subjdir}.bedpostX/temp
	torque_command="qsub -V $queue -l nodes=1:ppn=1:gpus=1,walltime=00:40:00 -N bedpostx_postproc_gpu -o ${subjdir}.bedpostX/logs -e ${subjdir}.bedpostX/logs -W depend=afterokarray:$bedpostid"
	mergeid=`exec $torque_command ${subjdir}.bedpostX/temp | awk '{print $1}' | awk -F. '{print $1}'`
        rm ${subjdir}.bedpostX/temp
	sleep 10
fi

echo $mergeid > ${subjdir}.bedpostX/logs/postproc_ID

if [ "x$SGE_ROOT" != "x" ]; then
    echo
    echo Type ${subjdir}.bedpostX/monitor to show progress.
    echo Type ${subjdir}.bedpostX/cancel to terminate all the queued tasks.
    cat <<EOC > ${subjdir}.bedpostX/cancel
#!/bin/sh
qdel $mergeid $bedpostid
EOC
    chmod +x ${subjdir}.bedpostX/cancel

    echo
    echo You will get an email at the end of the post-processing stage.
    echo

elif [ "x$FSLPARALLEL" != "x" ] && [ $slurm -eq 1 ]; then
        echo
        echo Type ${subjdir}.bedpostX/monitor to show progress.
        echo Type ${subjdir}.bedpostX/cancel to terminate all the queued tasks.
        cat <<EOC > ${subjdir}.bedpostX/cancel
#!/bin/sh
scancel $mergeid
scancel $bedpostid
scancel $preprocid
EOC
        chmod +x ${subjdir}.bedpostX/cancel

        echo
        echo You will get an email at the end of the post-processing stage.
        echo

else
    sleep 60
fi
