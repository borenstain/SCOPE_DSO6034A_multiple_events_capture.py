
Script captures electronic signal of a randomly occurring multiple events using a single channel Agilent made DSO6034A oscilloscope. 
Get the data into an array, scales and saves it in a csv file. Name of csv file is the date and time at the start of the test.
The 1st column of the csv file is the time vector, in seconds. At the beginning of the data of each event, there is a time stamp from the
beginning of the run in seconds.
The RUN/STOP of scope need to be in red.
The trigger should be set above the maximum regularly occuring signal (seam).
Or, it should be set at the two or three times the standard deviation of the background signal.
It is possible to change trigger or scale of chnnel while running to adjust the signal into scale.
That is because programs gets scaling information so called "pre_amble" for every event.
NO ERROR CHECKING OR HANDLING IS INCLUDED
Based on InfiniiVision_Analog_Waveform_Grab_Elegant.py
