# -*- coding: utf-8 -*-
# Updated for Python 3.7 on December 26 2019
# Written by Shmuel Borenstain
# Requires VISA installed on Control PC
# 'keysight.com/find/iosuite'
# Requires PyVisa to use VISA in Python
# 'http://PyVisa.sourceforge.net/PyVisa/'
## Keysight IO Libraries 17.1.19xxx
## PyVisa 1.8
###############################################################################
## Description
# Based on InfiniiVision_Analog_Waveform_Grab_Elegant.py
## Script captures signal of a randomly occuring multiple events using a single 
## channel Agilent made DSO6034A oscilloscope##. 
## Get the data into an array, scales and saves it in a csv file. 
## Name of file is the date and time at the start of the test.
## At the beginning of the data of each event, there is a time stamp from the
## beginning of the run.
## The RUN/STOP of scope need to be in red.
## The trigger should be set above the maximum regularly occuring signal (seam).
## Or, it should be set at the two or three times the standard deviation of the background signal.
## It is possible to change trigger or scale of chnnel while running to adjust the signal into scale.
## That is because programs gets scaling information so called "pre_amble" for every event.
## NO ERROR CHECKING OR HANDLING IS INCLUDED
## ALWAYS DO SOME TEST RUNS!!!!! and ensure you are getting what you want and it is later usable!!!!!
###############################################################################
## Import Python modules
##############################################################################
import os
import sys
import visa # PyVisa info @ http://PyVisa.readthedocs.io/en/stable/
import time
# import tkinter as tk
import numpy as np
import matplotlib.pyplot as plt
from datetime import datetime
###############################################################################
## DEFINE CONSTANTS
###############################################################################
BASE_DIRECTORY = os.path.dirname(r'C:\Users\borensta\Documents\work-horse\data_from_scope\..') 
number_of_events = 5 ## Number of events to capture.
ch = 1 # Channel number used.
delay_ = 1e-3        ## pause in seconds after plot. zero is possible but will not show plot 
GLOBAL_TOUT =  60 # IO time out in seconds
USER_REQUESTED_POINTS = 1000
    ## Setting this to 8000000 or more will ensure that the maximum number of available points is retrieved, though often less will come back.
    ## Average and High Resolution acquisition types have shallow memory depth, and thus acquiring waveforms in Normal acq. type and post processing for High Res. or repeated acqs. for Average is suggested if more points are desired.
    ## Asking for zero (0) points, a negative number of points, fewer than 100 points, or a non-integer number of points (100.1 -> error, but 100. or 100.0 is ok) will result in an error, specifically -222,"Data out of range"
## Set Global Timeout
## This can be used wherever, but local timeouts are used for Arming, Triggering, and Finishing the acquisition... Thus it mostly handles IO timeouts
##############################################################################
## Initialization 
SCOPE_VISA_ADDRESS = "USB0::0x0957::0x1734::MY44001826::0::INSTR" 
    ## Note: USB transfers are generally fastest.
    ## Video: Connecting to Instruments Over LAN, USB, and GPIB in Keysight Connection Expert: https://youtu.be/sZz8bNHX5u4
## Save Locations
## IMPORTANT NOTE:  This script WILL overwrite previously saved files!
###############################################################################
sys.stdout.write("Script is running. This may take a while...")
##############################################################################
## Connect and initialize scope
###############################################################################
## Define VISA Resource Manager & Install directory using PyVisa
rm = visa.ResourceManager('C:\\Windows\\System32\\visa32.dll') # this uses PyVisa
## This is more or less ok too: rm = visa.ResourceManager('C:\\Program Files (x86)\\IVI Foundation\\VISA\\WinNT\\agvisa\\agbin\\visa32.dll')
## In fact, it is generally not needed to call it explicitly: rm = visa.ResourceManager()
## Define & open the scope by the VISA address 
try:
    KsInfiniiVisionX = rm.open_resource(SCOPE_VISA_ADDRESS)
except Exception:
    print("Unable to connect to oscilloscope at " + str(SCOPE_VISA_ADDRESS) + ". Aborting script.\n")
    sys.exit()
## Clear the instrument bus
KsInfiniiVisionX.clear()
KsInfiniiVisionX.timeout = GLOBAL_TOUT*1000
###############################################################################
## Setup data export - For repetitive acquisitions, this only needs to be done once unless settings are changed
KsInfiniiVisionX.write(":WAVeform:POINts:MODE MAX") 
KsInfiniiVisionX.write(":WAVeform:FORMat WORD") # 16 bit word format... or BYTE for 8 bit format - WORD recommended, see more comments below when the data is actually retrieved
    ## WORD format especially  recommended  for Average and High Res. Acq. Types, which can produce more than 8 bits of resolution.
KsInfiniiVisionX.write(":WAVeform:BYTeorder LSBFirst") # Explicitly set this to avoid confusion - only applies to WORD FORMat
KsInfiniiVisionX.write(":WAVeform:UNSigned 0") # Explicitly set this to avoid confusion
## Pre-allocate holders for the vertical Pre-ambles and Channel units
ANALOGVERTPRES = np.zeros([12])
## For readability: ANALOGVERTPRES = (Y_INCrement_Ch1, Y_INCrement_Ch2, Y_INCrement_Ch3, Y_INCrement_Ch4, Y_ORIGin_Ch1, Y_ORIGin_Ch2, Y_ORIGin_Ch3, Y_ORIGin_Ch4, Y_REFerence_Ch1, Y_REFerence_Ch2, Y_REFerence_Ch3, Y_REFerence_Ch4)
###############################################################################
## User should insert here delay_ between samples and number of samples:
###############################################################################
time_ref = time.perf_counter()
for i_event in range(0,number_of_events):
    KsInfiniiVisionX.write(":SINGle")
###############################################################################
    On_Off = int(KsInfiniiVisionX.query(":CHANnel" + str(ch) + ":DISPlay?")) # Is the channel displayed? If not, don't pull.
    if On_Off == 1: # Only ask if needed... but... the scope can acquire waveform data even if the channel is off (in some cases) - so modify as needed
        Channel_Acquired = int(KsInfiniiVisionX.query(":WAVeform:SOURce CHANnel" + str(ch) + ";POINts?")) # If this returns a zero, then this channel did not capture data and thus there are no points
    else:
        Channel_Acquired = 0
    if Channel_Acquired == 0 or On_Off == 0: # Channel is off or no data acquired
        KsInfiniiVisionX.clear()
        KsInfiniiVisionX.close()
        sys.exit("No data has been acquired. Properly closing scope and aborting script.")
############################################
    else: # Channel is on AND data acquired
        Pre = KsInfiniiVisionX.query(":WAVeform:PREamble?").split(',') # ## The programmer's guide has a very good description of this, under the info on :WAVeform:PREamble.
        ## In above line, the waveform source is already set; no need to reset it.
        ANALOGVERTPRES[ch-1]  = float(Pre[7]) # Y INCrement, Voltage difference between data points; Could also be found with :WAVeform:YINCrement? after setting :WAVeform:SOURce
        ANALOGVERTPRES[ch+3]  = float(Pre[8]) # Y ORIGin, Voltage at center screen; Could also be found with :WAVeform:YORigin? after setting :WAVeform:SOURce
        ANALOGVERTPRES[ch+7]  = float(Pre[9]) # Y REFerence, Specifies the data point where y-origin occurs, always zero; Could also be found with :WAVeform:YREFerence? after setting :WAVeform:SOURce
        ## In most cases this will need to be done for each channel as the vertical scale and offset will differ. However,
        ## if the vertical scales and offset are identical, the values for one channel can be used for the others.
        ## For math waveforms, this should always be done.
        ## CH_UNITS[ch-1] = str(KsInfiniiVisionX.query(":CHANnel" + str(ch) + ":UNITs?").strip('\n')) # This isn't really needed but is included for completeness
    del On_Off, Channel_Acquired
#####################################################################################################################################
## Set and get points to be retrieved - For repetitive acquisitions, this only needs to be done once unless scope settings are changed
## This is non-trivial, but the below algorithm always works w/o throwing an error, as long as USER_REQUESTED_POINTS is a positive whole number (positive integer)
#########################################################
## Determine Acquisition Type to set points mode properly
    ACQ_TYPE = str(KsInfiniiVisionX.query(":ACQuire:TYPE?")).strip("\n")
            ## This can also be done when pulling pre-ambles (pre[1]) or may be known ahead of time, but since the script is supposed to find everything, it is done now.
    if ACQ_TYPE == "AVER" or ACQ_TYPE == "HRES": # Don't need to check for both types of mnemonics like this: if ACQ_TYPE == "AVER" or ACQ_TYPE == "AVERage": becasue the scope ALWAYS returns the short form
        POINTS_MODE = "NORMal" # Use for Average and High Resoultion acquisition Types.
            ## If the :WAVeform:POINts:MODE is RAW, and the Acquisition Type is Average, the number of points available is 0. If :WAVeform:POINts:MODE is MAX, it may or may not return 0 points.
            ## If the :WAVeform:POINts:MODE is RAW, and the Acquisition Type is High Resolution, then the effect is (mostly) the same as if the Acq. Type was Normal (no box-car averaging).
            ## Note: if you use :SINGle to acquire the waveform in AVERage Acq. Type, no average is performed, and RAW works. See sample script "InfiniiVision_2_Simple_Synchronization_Methods.py"
    else:
        POINTS_MODE = "RAW" # Use for Acq. Type NORMal or PEAK
        ## Note, if using "precision mode" on 5/6/70000s or X6000A, then you must use POINTS_MODE = "NORMal" to get the "precision record."
    ## Note:
        ## :WAVeform:POINts:MODE RAW corresponds to saving the ASCII XY or Binary data formats to a USB stick on the scope
        ## :WAVeform:POINts:MODE NORMal corresponds to saving the CSV or H5 data formats to a USB stick on the scope
    ###########################################################################################################
    ## Find max points for scope as is, ask for desired points, find how many points will actually be returned
        ## KEY POINT: the data must be on screen to be retrieved.  If there is data off-screen, :WAVeform:POINts? will not "see it."
            ## Addendum 1 shows how to properly get all data on screen, but this is never needed for Average and High Resolution Acquisition Types,
            ## since they basically don't use off-screen data; what you see is what you get.
    KsInfiniiVisionX.write(":WAVeform:SOURce CHANnel" + str(ch))
    ## The next line is similar to, but distinct from, the previously sent command ":WAVeform:POINts:MODE MAX".  This next command is one of the most important parts of this script.
    KsInfiniiVisionX.write(":WAVeform:POINts MAX") # This command sets the points mode to MAX AND ensures that the maximum # of points to be transferred is set, though they must still be on screen
    ## Since the ":WAVeform:POINts MAX" command above also changes the :POINts:MODE to MAXimum, which may or may not be a good thing, so change it to what is needed next.
    KsInfiniiVisionX.write(":WAVeform:POINts:MODE " + str(POINTS_MODE))
    ## If measurements are also being made, they are made on the "measurement record."  This record can be accessed by using:
        ## :WAVeform:POINts:MODE NORMal instead of :WAVeform:POINts:MODE RAW
        ## Please refer to the progammer's guide for more details on :WAV:POIN:MODE RAW/NORMal/MAX
    ## Now find how many points are actually currently available for transfer in the given points mode (must still be on screen)
    MAX_CURRENTLY_AVAILABLE_POINTS = int(KsInfiniiVisionX.query(":WAVeform:POINts?")) # This is the max number of points currently available - this is for on screen data only - Will not change channel to channel.
    ## The scope will return a -222,"Data out of range" error if fewer than 100 points are requested, even though it may actually return fewer than 100 points.
    if USER_REQUESTED_POINTS < 100:
        USER_REQUESTED_POINTS = 100
    ## One may also wish to do other tests, such as: is it a whole number (integer)?, is it real? and so forth...
    if MAX_CURRENTLY_AVAILABLE_POINTS < 100:
       MAX_CURRENTLY_AVAILABLE_POINTS = 100
    if USER_REQUESTED_POINTS > MAX_CURRENTLY_AVAILABLE_POINTS or ACQ_TYPE == "PEAK":
         USER_REQUESTED_POINTS = MAX_CURRENTLY_AVAILABLE_POINTS
         ## Note: for Peak Detect, it is always suggested to transfer the max number of points available so that narrow spikes are not missed.
         ## If the scope is asked for more points than :ACQuire:POINts? (see below) yields, though, not necessarily MAX_CURRENTLY_AVAILABLE_POINTS, it will throw an error, specifically -222,"Data out of range"
    ## If one wants some other number of points...
    ## Tell it how many points you want
    KsInfiniiVisionX.write(":WAVeform:POINts " + str(USER_REQUESTED_POINTS))
    ## Then ask how many points it will actually give you, as it may not give you exactly what you want.
    NUMBER_OF_POINTS_TO_ACTUALLY_RETRIEVE = int(KsInfiniiVisionX.query(":WAVeform:POINts?"))
    Pre = KsInfiniiVisionX.query(":WAVeform:PREamble?").split(',') # This does need to be set to a channel that is on, but that is already done... e.g. Pre = KsInfiniiVisionX.query(":WAVeform:SOURce CHANnel" + str(FIRST_CHANNEL_ON) + ";PREamble?").split(',')
    ## While these values can always be used for all analog channels, they need to be retrieved and used separately for math/other waveforms as they will likely be different.
    #ACQ_TYPE    = float(Pre[1]) # Gives the scope Acquisition Type; this is already done above in this particular script
    X_INCrement = float(Pre[4]) # Time difference between data points; Could also be found with :WAVeform:XINCrement? after setting :WAVeform:SOURce
    X_ORIGin    = float(Pre[5]) # Always the first data point in memory; Could also be found with :WAVeform:XORigin? after setting :WAVeform:SOURce
    X_REFerence = float(Pre[6]) # Specifies the data point associated with x-origin; The x-reference point is the first point displayed and XREFerence is always 0.; Could also be found with :WAVeform:XREFerence? after setting :WAVeform:SOURce
    ##
    del Pre
    ##
    DataTime = ((np.linspace(0,NUMBER_OF_POINTS_TO_ACTUALLY_RETRIEVE-1,NUMBER_OF_POINTS_TO_ACTUALLY_RETRIEVE)-X_REFerence)*X_INCrement)+X_ORIGin
    if ACQ_TYPE == "PEAK": # This means Peak Detect Acq. Type
        DataTime = np.repeat(DataTime,2)
        ##  The points come out as Low(time1),High(time1),Low(time2),High(time2)....
        ### SEE IMPORTANT NOTE ABOUT PEAK DETECT AT VERY END, specific to fast time scales
    
    ###############################################################################
    ## Pre-allocate data array 
    if ACQ_TYPE == "PEAK": # This means peak detect mode ### SEE IMPORTANT NOTE ABOUT PEAK DETECT MODE AT VERY END, specific to fast time scales
        Wav_Data = np.zeros([2*NUMBER_OF_POINTS_TO_ACTUALLY_RETRIEVE])
        ## Peak detect mode returns twice as many points as the points query, one point each for LOW and HIGH values
    else: # For all other acquistion modes
        Wav_Data = np.zeros([NUMBER_OF_POINTS_TO_ACTUALLY_RETRIEVE])
    ###############################################################################
    ## Determine number of bytes that will actually be transferred and set the "chunk size" accordingly.
        ## When using PyVisa, this is in fact completely unnecessary, but may be needed in other leagues, MATLAB, for example.
        ## However, the benefit in Python is that the transfers can take less time, particularly longer ones.
    ## Get the waveform format
    WFORM = str(KsInfiniiVisionX.query(":WAVeform:FORMat?"))
    if WFORM == "BYTE":
        FORMAT_MULTIPLIER = 1
    else: #WFORM == "WORD"
        FORMAT_MULTIPLIER = 2
    
    if ACQ_TYPE == "PEAK":
        POINTS_MULTIPLIER = 2 # Recall that Peak Acq. Type basically doubles the number of points.
    else:
        POINTS_MULTIPLIER = 1  
    TOTAL_BYTES_TO_XFER = POINTS_MULTIPLIER * NUMBER_OF_POINTS_TO_ACTUALLY_RETRIEVE * FORMAT_MULTIPLIER + 11
        ## Why + 11?  The IEEE488.2 waveform header for definite length binary blocks (what this will use) consists of 10 bytes.  The default termination character, \n, takes up another byte.
            ## If you are using mutliplr termination characters, adjust accordingly.
        ## Note that Python 2.7 uses ASCII, where all characters are 1 byte.  Python 3.5 uses Unicode, which does not have a set number of bytes per character.
    ## Set chunk size:
        ## More info @ http://pyvisa.readthedocs.io/en/stable/resources.html
    if TOTAL_BYTES_TO_XFER >= 400000:
        KsInfiniiVisionX.chunk_size = TOTAL_BYTES_TO_XFER
    ###############################################################################
    ## Pull waveform data, scale it
    ## Gets the waveform in 16 bit WORD format
    Wav_Data[:] = np.array(KsInfiniiVisionX.query_binary_values(':WAVeform:SOURce CHANnel' + str(ch) + ';DATA?', "h", False)) 
    Wav_Data[:] = ((Wav_Data[:]-ANALOGVERTPRES[ch+7])*ANALOGVERTPRES[ch-1])+ANALOGVERTPRES[ch+3]
    ## For clarity: Scaled_waveform_Data[*] = [(Unscaled_Waveform_Data[*] - Y_reference) * Y_increment] + Y_origin
    ## Reset the chunk size back to default if needed.
    if TOTAL_BYTES_TO_XFER >= 400000:
        KsInfiniiVisionX.chunk_size = 20480
        ## If you don't do this, and now wanted to do something else... such as ask for a measurement result, and leave the chunk size set to something large,
        ## it can really slow down the script, so set it back to default, which works well.
    ###################################################################
    now=datetime.now()
    time_stamp = time.perf_counter() - time_ref 
    if i_event == 0:
        plt.plot(DataTime,Wav_Data)
         # plt.title('Fantastic Plot #1')
        plt.xlabel('Time')
        plt.ylabel('Volts')
        plt.pause(0.15)  ## 0.1 does not provide plot.
        time_stamped_filename = now.strftime("%Y_%m_%d_%H_%M_%S")
        accumulated_scope_data=np.vstack((np.concatenate(([time_stamp],DataTime)),np.concatenate(([time_stamp],Wav_Data))))
        print("Event ",i_event," captured.")
    else:
        plt.plot(DataTime,Wav_Data)
        plt.pause(delay_)
        accumulated_scope_data=np.vstack((accumulated_scope_data,np.concatenate(([time_stamp],Wav_Data))))  
        print("Event # ",i_event," captured.")
########################################################
## Saving as CSV - Note: first raw is the time stamp in seconds from start of script.
########################################################
filename = os.path.abspath(os.path.join(BASE_DIRECTORY,time_stamped_filename+".csv"))
with open(filename, 'w') as filehandle: # w means open for writing; can overwrite
    np.savetxt(filehandle, np.transpose(accumulated_scope_data), delimiter=',')
###################################################################
    ## Done with scope operations - Close Connection to scope properly    
###################################################################
KsInfiniiVisionX.clear()
KsInfiniiVisionX.close()
del KsInfiniiVisionX
