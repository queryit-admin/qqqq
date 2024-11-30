'use client';

import { XMarkIcon } from '@heroicons/react/24/outline';
import { useEffect, useState } from 'react';
import { createPortal } from 'react-dom';
import { api, endpoints } from '@/lib/api';

interface NodeDescriptionModalProps {
  nodeId: string;
  isOpen: boolean;
  onClose: () => void;
}

export default function NodeDescriptionModal({ nodeId, isOpen, onClose }: NodeDescriptionModalProps) {
  const [description, setDescription] = useState<string>('');
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [mounted, setMounted] = useState(false);

  useEffect(() => {
    setMounted(true);
    return () => setMounted(false);
  }, []);

  useEffect(() => {
    if (isOpen && nodeId) {
      fetchNodeDescription();
    }
  }, [isOpen, nodeId]);

  const fetchNodeDescription = async () => {
    try {
      setIsLoading(true);
      setError(null);
      const response = await api.get(endpoints.nodes.description(nodeId));
      setDescription(response.data.description_html);
    } catch (err) {
      console.error('Error fetching node description:', err);
      setError('Failed to load node description');
    } finally {
      setIsLoading(false);
    }
  };

  if (!isOpen || !mounted) return null;

  const modal = (
    <div className="fixed inset-0 z-[9999] overflow-y-auto" aria-labelledby="modal-title" role="dialog" aria-modal="true">
      {/* Backdrop */}
      <div 
        className="fixed inset-0 bg-black bg-opacity-50 transition-opacity backdrop-blur-sm" 
        onClick={onClose}
      />

      {/* Modal */}
      <div className="flex min-h-screen items-center justify-center p-4">
        <div className="relative transform overflow-hidden rounded-xl bg-white shadow-2xl transition-all w-full max-w-4xl">
          {/* Close button */}
          <button
            onClick={onClose}
            className="absolute right-4 top-4 rounded-lg p-2 hover:bg-gray-100 transition-colors"
            aria-label="Close modal"
          >
            <XMarkIcon className="h-6 w-6 text-gray-500" />
          </button>

          {/* Content */}
          <div className="bg-white">
            {isLoading ? (
              <div className="p-8">
                <div className="animate-pulse space-y-6">
                  <div className="h-10 w-3/4 bg-gray-200 rounded-lg"></div>
                  <div className="h-40 bg-gray-200 rounded-lg"></div>
                  <div className="h-32 bg-gray-200 rounded-lg"></div>
                </div>
              </div>
            ) : error ? (
              <div className="p-8">
                <div className="text-red-500 text-center font-medium">{error}</div>
              </div>
            ) : (
              <div 
                dangerouslySetInnerHTML={{ __html: description }}
                className="max-h-[80vh] overflow-y-auto"
              />
            )}
          </div>
        </div>
      </div>
    </div>
  );

  // Use createPortal to render the modal at the document root
  return createPortal(modal, document.body);
}
